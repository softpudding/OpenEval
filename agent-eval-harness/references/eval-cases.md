# Building and growing the evaluation dataset

A case is a unit of evaluation:

```yaml
id: refund_unverified_user
initial_state:
  db_fixture: fixtures/orders_basic.sql
  user_session: anonymous
input: "Hi, I want a refund for order 5523."
expected:
  must_call: [verify_identity]
  must_not_call: [issue_refund]
  rubric:
    tone_appropriate: "polite, explains why ID verification is needed"
```

Keep cases as data, not code. JSONL or YAML, in the repo, reviewed in PRs.

## Start small, span failures

Day 1: 5–20 cases. Hand-crafted. Each one should target a **specific failure mode** the user has actually seen or worries about. A starter set might be:

- 1–2 happy-path cases (so the harness can show green).
- 3–5 cases for the failure modes the user named in Phase 1.
- 1–2 edge cases (empty input, very long input, conflicting input, hostile input).
- 1 case the user expects to be hard but doesn't want the agent to fail on silently.

If the user can't articulate 5 distinct failure modes, the agent isn't well-understood yet — that's a problem the harness can't fix on its own. Surface this to the user; don't paper over it with synthetic cases.

## Growing the dataset

Three sources, in roughly increasing order of value:

### 1. Synthetic generation from templates

Useful for combinatorial coverage: same task shape with varied parameters. Write a generator, not a model — model-generated cases drift toward what the model finds plausible, not what the agent will actually see. Keep generated cases in a separate file from hand-crafted ones; mark them as synthetic.

### 2. Mining from issue trackers / support tickets / chat logs

Pull real complaints, real questions, real failure reports. Each becomes a candidate case after the team writes an expected outcome.

### 3. Production traffic replay (highest value)

Once the agent has real users, instrument production to log inputs + env snapshots (anonymized). Convert a sample weekly into cases. This is the loop that keeps the harness aligned with reality (Principle 1).

For replay:
- Sample by failure: cases the team or users flagged as bad are highest signal.
- Sample by stratum: don't only mine failures — include a fraction of high-volume happy cases so regressions get caught.
- Anonymize aggressively: names, IDs, free-text fields. PII in fixtures is a security problem.
- Pin a version of the env state when the case was captured. Production schemas drift; replays must too.

## Dataset versioning

Treat the dataset like a benchmark version. When cases are added or expected outcomes change, bump a version. Old runs were scored against the old dataset; comparing across versions requires care.

A simple convention:

```
tasks/
  v1/
    cases.jsonl
    fixtures/
  v2/
    cases.jsonl
    fixtures/
  CHANGELOG.md
```

## Hard cases earn their keep

A case that the agent always passes carries no signal. A case that always fails carries no signal once it's logged. The cases that matter are the ones near the agent's current ability frontier — where small changes flip the score. Over time, retire trivially-passing cases (or move them to a smoke set) and invest in harder ones.

## Anti-patterns

- **Curator overlap.** The person who wrote the agent also writes all the cases. Inevitable blind spots. Have someone else write at least a third.
- **Score-driven dataset edits.** Tweaking expected outcomes because the agent failed is how you build a harness the agent always passes. If a case is wrong, fix the case with a documented rationale. If the agent is wrong, fix the agent.
- **One giant case file.** Group cases by category. Makes failures aggregable and PR diffs reviewable.
