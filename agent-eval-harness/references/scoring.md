# Scoring approaches

Two families: **static** (programmatic) and **LLM-as-judge**. Most production harnesses use both — static for the checkable parts, LLM-judge for the open-ended parts.

## Default: prefer static

If the agent's output is a checkable artifact, score it programmatically. Static scorers are:
- Deterministic (same input → same score)
- Cheap (no model call per case)
- Auditable (you can read the scoring code)
- Objective (no model bias drift over time)

Examples of static-scorable outputs:
- "Did the SQL the agent wrote return the expected rows?" → run it, diff.
- "Did the agent call `create_order` with `item_id=X, qty=Y`?" → assert against the recorded tool calls.
- "Does the patch apply and do the repo's tests pass?" → SWE-bench.
- "Is the final file content equal to the expected file content?" → byte-diff or AST-diff.

Static scorers should return a structured result, never just a number:

```python
@dataclass
class Score:
    passed: bool
    value: float        # 0.0–1.0
    details: dict       # diagnostic info: diffs, missing fields, etc.
```

`details` is what gives the user Principle 2. Don't drop it.

## When static doesn't fit: LLM-as-judge with rubrics

Use LLM-judge when the output's quality is genuinely subjective: tone, helpfulness, summarization fidelity, safety, persuasiveness. **Never call a naive judge.** A naive judge ("rate this reply 1–10") is high-variance, biased toward verbosity, and tells you nothing about *why* it scored low.

### Rubric pattern

Define 3–6 dimensions per case (or per case category). For each dimension:
- A name (e.g. `tone_appropriate`, `policy_followed`, `addresses_question`).
- A short definition.
- Discrete levels with anchors. Prefer 0/1 or a small ordinal (0/1/2). Avoid 1–10 — judges cluster around 7.

The judge prompt should ask the model to:
1. Score each dimension independently.
2. Quote evidence from the agent's output for each score.
3. Refuse to score a dimension if evidence is absent (return `n/a`).

Store the full per-dimension breakdown plus the judge's reasoning. Aggregate later, in analysis, not at scoring time.

### Reducing judge noise

- **Pin the judge model and prompt.** Bumping either changes scores; treat it like a benchmark version change.
- **Few-shot the rubric.** Include 1–2 worked examples per dimension in the judge prompt.
- **Multiple samples for borderline cases.** If `score ∈ {0, 1}` flips across 3 samples, mark the case "ambiguous" rather than averaging.
- **Spot-check against humans.** Periodically (every N cases or every release) have a human re-grade a random sample; track judge-human agreement as a meta-metric.

### Where LLM-judge fails

- **Self-evaluation bias.** Same model evaluating itself scores too high. Use a different family if possible.
- **Length bias.** Long responses score higher absent length anchors. Add a `concise` dimension if relevant.
- **Prompt sensitivity.** Reordering rubric dimensions changes scores. Lock the prompt.

## Hybrid scoring (very common)

A realistic harness for, say, a customer-support agent:

| Dimension | Type | What it checks |
|---|---|---|
| `correct_action` | static | Did the agent call `refund_order(id=...)` with the right args? |
| `policy_followed` | static | Did the agent verify identity before issuing refund? (check trace for the verify step) |
| `tone_appropriate` | LLM-judge | Was the reply empathetic and professional? |
| `addresses_concern` | LLM-judge | Did the reply address the user's stated problem? |

Each dimension is scored independently. Aggregation (weighted average, all-must-pass, etc.) is a separate concern handled at analysis time.

## Don't score in a single number at scoring time

The number is for dashboards. The dimensions are for engineering. Keep them separate. Storing only the aggregate destroys Principle 2.
