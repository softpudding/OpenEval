# First principles for agent evaluation

Two principles. A harness that violates either will mislead the team building the agent.

## Principle 1 — The harness must measure what the agent is actually for

State the agent's purpose in one sentence. Then ask: "if a run gets a perfect score on this harness, is the user actually happy?" If the answer is "maybe", the harness is measuring a proxy, not the thing.

Common failure modes:
- **Proxy creep.** Started measuring "did the agent call the right API", drifted into "did the response contain the word 'sure'". Now the score correlates with verbosity, not correctness.
- **Easy-case bias.** Dataset was hand-built by the person who wrote the agent. It only covers the cases the agent was designed for. Score is high; production performance is low.
- **Mismatched grain.** Agent's job is to "complete a multi-step order"; harness checks "did the final message look like a confirmation". An agent that hallucinates a confirmation without doing the order will score full marks.

Concrete check: write down 3 ways a real user could be unhappy with the agent in production. For each, can the harness catch it? If not, the harness fails Principle 1.

## Principle 2 — The harness must localize failures

A single aggregate score tells you the agent is bad. It does not tell you *why*. A good harness produces per-case, per-dimension, ideally per-step output so the team can prioritize what to fix next.

Practical implications:
- **Don't collapse rubrics early.** If an LLM judge evaluates 4 dimensions, store all 4. The aggregate is a view, not the data.
- **Capture context, not just verdicts.** For each failing case, the record should include the input, the agent's output, the expected outcome, and the scorer's reasoning. Diffing these is how regressions get fixed.
- **Group failures by type, not just by case.** "12 cases failed because the tool call was missing a required arg" is actionable. "12 cases scored < 0.5" is not.
- **Trace evaluation, when enabled, exists for this principle.** See `result-vs-trace.md`.

Concrete check: a teammate looks at the harness output for a bad run. Within 60 seconds, can they answer "what should I change in the agent?" If not, the harness fails Principle 2.

## Why these two

Principle 1 is about validity — the harness measures the right thing.
Principle 2 is about actionability — the measurement leads to a fix.

A harness that's valid but not actionable becomes a number on a dashboard that nobody knows how to move. A harness that's actionable but not valid drives the agent confidently in the wrong direction. Both fail.
