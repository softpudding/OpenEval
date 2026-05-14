# Designing the simulation environment

The agent acts somewhere — a codebase, a browser, a database, a chat session, an API surface. The harness needs a stand-in for that "somewhere" that is **reproducible, isolated, and realistic**.

## Three requirements

1. **Reproducible** — running case `C` twice yields the same starting state. No leaked side effects from the previous case. No drift from the outside world.
2. **Isolated** — the agent cannot reach beyond the harness. No prod DB writes, no real Stripe charges, no emails sent to real addresses.
3. **Realistic** — close enough to production that an agent that passes in the harness will plausibly pass in prod. Gaps between sim and prod are the #1 source of "we shipped and it broke".

These three pull against each other. Reproducibility favors mocks; realism favors real systems. Pick the leanest setup that gets you "close enough" for the failure modes the user cares about.

## Common patterns by agent shape

| Agent shape | Recommended env |
|---|---|
| Coding agent | Per-case ephemeral Docker container with the repo checked out at a pinned SHA. SWE-bench style. |
| Web/UI agent | Headless browser (Playwright) pointed at a fixture site or a staging deploy with seeded data. |
| API/tool-calling agent | In-process fake of each tool, returning canned responses indexed by request. Optionally record/replay against real APIs in "golden" mode. |
| Chatbot / advisor | Stateless — env is just the conversation history + any retrieved docs. Pin the retrieval corpus. |
| OS/desktop agent | VM snapshot per case (OSWorld style). Heaviest setup; only when nothing lighter captures the behavior. |
| DB-mutating agent | Ephemeral SQLite or per-case Postgres schema, seeded from SQL fixture before each case. |

## The reset contract

Every env adapter must support:

```python
def reset(case) -> EnvHandle: ...
def teardown(handle: EnvHandle) -> None: ...
```

`reset` must put the env into the exact state defined by the case's `initial_state`. `teardown` must release resources and guarantee no leakage to the next case. If reset is expensive (e.g. VM snapshot restore), consider per-batch reset with isolation guarantees, but understand the tradeoff: a flaky case can poison the rest of the batch.

## Recording real production state into the env

Real production cases are gold for the harness (Principle 1). To convert one:

1. Capture the agent's input and the relevant slice of env state at request time.
2. Strip PII; anonymize identifiers.
3. Freeze it as a fixture (SQL dump, JSON snapshot, recorded HTTP cassette).
4. Add an expected outcome — either the action the agent *should* have taken, or a rubric for grading.

This is the loop that makes the harness keep up with prod. See `eval-cases.md`.

## Anti-patterns

- **Sharing env state across cases.** Score becomes order-dependent. Hide bugs from yourself.
- **Letting the env reach the real internet.** Flaky and non-reproducible. Use VCR-style cassettes.
- **Building the env to match what the agent currently does.** Build it to match production. If the agent fails, that's the harness working.
- **Skipping teardown.** Disk fills up, ports leak, next case fails for unrelated reasons.
