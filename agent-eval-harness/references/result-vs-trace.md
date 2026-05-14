# Result-only vs trace evaluation

A choice about *what* the harness inspects. Not mutually exclusive — many production harnesses do both.

## Result-only

Score whatever the agent produced at the end: final answer, final filesystem state, the tool call it ultimately made. The trace is captured (for debugging) but not scored.

**When to choose:**
- The agent's job is defined by an end-state (file written, order placed, PR merged).
- The team currently ships a number, not a fix list.
- Most public benchmarks (SWE-bench, OSWorld, WebArena) work this way — the canonical pattern.

**Strengths:** simple, cheap, hard to game, easy to compare across agent versions.

**Weakness:** a failure tells you the run failed, not where. If the agent took 14 steps and step 7 was where it went wrong, result-only scoring doesn't help you find step 7.

## Trace evaluation

Score the sequence of agent steps. Each step is `(thought, action, observation)` or similar. Scorers can inspect any step.

Two flavors:

1. **Step-level checks.** Specific cases assert specific steps. "By step ≤3, the agent must have called `lookup_user`." Cheap, deterministic, but only catches what you wrote rules for.
2. **Trace-level rubric (LLM judge).** Give a model the full trace, ask rubric questions like "did the agent ever invent a parameter that wasn't in the input?", "did it give up prematurely?". Expensive but flexible.

**When to choose:**
- During Phase 1, the user said something like "I want to know *where* it goes wrong", or "the final answer is fine but I think it's getting there by luck."
- The agent is multi-step (more than ~3 actions) and steps are observable.
- The team is actively iterating on the agent's policy / prompts / tools and needs signal per change.

**Strengths:** localizes failure, exposes anti-patterns the result-only view hides (e.g. agent solves the task but takes 40 steps when 4 suffice).

**Weakness:** more code, more cost, more places for the harness itself to be wrong. Defining "good trace" is harder than "good result".

## Doing both (recommended for production)

Score the result for the leaderboard number. Annotate the trace for diagnosis. Failures get both: "FAIL: final state mismatch; trace annotation: agent skipped the verification step at step 4."

A useful trace schema for the harness:

```python
@dataclass
class TraceStep:
    index: int
    kind: Literal["thought", "tool_call", "tool_result", "message"]
    payload: dict          # tool name + args, message text, etc.
    timestamp: float
```

Keep the schema flat and serializable. Don't bake in agent-framework specifics — write a thin shim from the agent's native trace format to this one.

## A failure mode to avoid

Scoring the trace *instead of* the result is a trap. An agent that follows the "right" steps but produces the wrong final state is failing — give it a 0 even if the trace looks clean. Trace scoring is a diagnostic *over* result scoring, not a substitute.
