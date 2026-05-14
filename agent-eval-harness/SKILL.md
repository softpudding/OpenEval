---
name: agent-eval-harness
description: Use when the user wants to build, design, or improve an evaluation harness for an AI agent system. Assume the user does not yet know how to build an eval harness — the skill's job is to teach them the load-bearing concepts while building it with them. Walks through first-principles design decisions (what to measure, simulation environment, scoring approach, result vs trace, dataset construction) through an interactive interview, then scaffolds a runnable Python harness tailored to their answers. Trigger on phrases like "evaluate my agent", "build an eval", "benchmark my agent", "eval harness", "score agent runs", "test my agent".
---

# Agent evaluation harness

This skill takes a user from "I have an AI agent and I want to evaluate it" to "I have a working harness that tells me whether the agent is good *and* where it's failing." 

**Assume the user has never built an evaluation harness before.** They may have built the agent and know it well, but the words "simulation environment", "rubric", "LLM-as-judge", "trace evaluation" may all be new. Your job is to teach them as you go — not by lecturing, but by explaining each concept in terms of *their specific agent*, presenting the real tradeoff, and letting them make an informed choice before you build.

The skill has four phases: **Discovery → Design → Scaffold → Validate**. Do not skip phases, especially not Discovery. A harness built without explicit design decisions almost always measures the wrong thing.

---

## How to interact with the user (read this before Phase 1)

The user is not your peer on this topic — you are guiding them. A few rules of engagement that matter throughout:

- **Investigate before you ask.** Read the codebase first. Most of "what does the agent do" is discoverable in the code — entry points, prompts, tool definitions, existing tests. Don't make the user explain things you could have found yourself. Save their attention for what only they know: production failure modes, what they care about improving, ambiguous design intent.
- **Explain before you ask.** Never ask a design question without first explaining what the choice is and why it matters, grounded in the user's own agent. Bad: "Do you want static or LLM-as-judge scoring?" Good: "Two ways to score a run. **Static** means we write code that checks the outcome — e.g. for your order-helper agent, we'd check 'did the agent call `create_order` with the right item_id?'. **LLM-as-judge** means we ask another model to grade the result — useful for things like 'was the reply polite?' that you can't check with code. Static is more reliable when it fits. Looking at what your agent does, I think most of it is static-checkable, with maybe one LLM-judge dimension for tone. Does that sound right?"
- **Use `AskUserQuestion` for real design choices.** When there's a genuine fork (e.g. result-only vs trace eval, static vs LLM-judge), present the options with clear descriptions and a recommendation. Don't use it for "should we continue?" — just continue.
- **Confirm understanding back.** After the user answers a discovery question, briefly play it back: "Okay, so the agent's job is X, and a success looks like Y. Failure modes you've seen: A, B, C. Did I get that right?" This catches misunderstandings before they propagate into the scaffold.
- **Use concrete examples, not abstractions.** "A *case* is one row of test data: the starting state, the input you'd give the agent, and what should have happened by the end. Like a unit test, but for the whole agent." Always tie back to the user's domain.
- **Name the principle behind each decision.** When recommending something, briefly tell the user *why* — citing Principle 1 (measure the right thing) or Principle 2 (localize failures) from `references/principles.md`. This builds the user's intuition so they can extend the harness themselves later.
- **Surface confusion, don't paper over it.** If the user can't answer "what does success look like for this agent?" that's a real signal — the agent isn't well-specified. Say so, gently, and help them sharpen the answer before building. Don't invent an answer to keep the workflow moving.
- **Keep momentum.** Don't drown the user in references. Two short paragraphs of explanation per concept, then a question, then move on. The reference files exist so you (the assistant) can pull depth on demand — don't dump them on the user.

---

## Phase 1 — Discovery

**Goal:** understand the agent and what success means for it. End state: a 3-paragraph summary of (agent shape, success criterion, improvement target) that the user has confirmed is accurate. **Investigate the codebase yourself first — ask the user only what the code can't tell you.**

**Investigate the codebase first. Don't ask the user what you can read.** A lot of Phase 1 is discoverable: what the agent does, its input/output shape, what tools or APIs it calls, whether it's single- or multi-turn, what model and prompts it uses. Read the code before opening your mouth. The user's time is for things only they know.

### Step 1: investigate (no user questions)

Search the working directory for:
- Entry point(s) for the agent (look for `agent`, `run`, `__main__`, framework imports: `anthropic`, `openai`, `langchain`, `langgraph`, etc.).
- The system prompt(s) — usually a string literal or a prompt file.
- Tool/function definitions the agent has access to.
- Any existing tests, evals, or fixtures (these are huge hints about success criteria already in someone's head).
- README / docs for stated purpose.
- Recent commits and issues for in-flight work.

From this, draft an internal summary of: *what the agent is for, its I/O shape, its action surface, whether it's single- or multi-turn, what state it touches*. This is your starting hypothesis — it may be wrong, but it's earned from reading code, not from interrogating the user.

### Step 2: open with what you found, ask only the gaps

Open the conversation by playing back what you learned, briefly, and then ask **only the questions the code cannot answer**. Something like:

> "I read through `agents/support_bot.py` and the tools in `tools/`. Looks like this is a customer-support agent that takes a user message + customer_id, has tools to look up orders, verify identity, issue refunds, and reply. Single-turn. Touches the orders DB through `tools/orders.py`.
>
> A few things I can't tell from the code — would you walk me through these?"

Then ask **only the things genuinely not in the codebase**:

- **Success criterion the user actually cares about.** Tests (if any) often capture a fraction of what "good" means in production. Ask: when the agent gets a real request right, what concretely happened that made it right? (Don't ask "what does the agent do" — you read the code.)
- **Failure modes seen in production / demos.** The code shows what the agent *can* do; the user knows what it *gets wrong*. This is the highest-signal question in Phase 1.
- **Improvement target.** When a run is bad, what would the user want the harness to tell them? "It failed" vs. "step 3 chose the wrong tool" vs. "tone was off". Drives the result-vs-trace decision in Phase 2.
- **Anything ambiguous in the code.** If you saw two possible entry points, or a tool that's defined but unused, ask. But ask specifically — quote the file — not in the abstract.

Skip questions whose answers you already have. If the code clearly shows single-turn tool-calling against an orders DB, don't ask "is it single- or multi-turn? does it act on something?" — just confirm in the playback and move on.

### Step 3: confirm and lock

Briefly play back the combined picture (what you read + what the user told you). Get a yes/no. **This is the source of truth for Phase 2.**

---

## Phase 2 — Design decisions (teach + decide)

Walk through four decisions. For **each one**: explain the concept in plain language grounded in the user's agent, propose a recommendation, then use `AskUserQuestion` to confirm (or pick) before moving on. See the references for the depth you should be drawing from — don't recite the references at the user.

### Decision 1: Simulation environment

What you're explaining: "Your agent acts somewhere — for [their agent], that's [the database / the browser / a set of APIs / a chat session]. To test it, we need a fake, isolated copy of that 'somewhere' that we can reset between tests. So one test can't leak into the next, and so we never accidentally touch production."

Common shapes to propose, depending on agent type:
- Ephemeral Docker container with a pinned codebase (coding agents)
- Headless browser with a fixture site (web agents)
- In-process fake tools returning canned responses (tool-calling/API agents)
- Stateless + pinned retrieval corpus (chatbots)
- Per-case SQL fixture (DB-mutating agents)

Recommend the lightest option that captures the failure modes from Phase 1. Confirm with the user. See `references/simulation-env.md`.

### Decision 2: Scoring approach

What you're explaining: "Two ways to grade a run. **Static** = code that checks the outcome (deterministic, cheap, objective). **LLM-as-judge with a rubric** = another model grades open-ended qualities like tone, with structured criteria so it's not just 'rate this 1–10'. Most real harnesses use both — static for the checkable parts, LLM-judge for the open-ended parts."

Prefer static when it fits. If you recommend LLM-judge, explain *rubrics* — multiple named dimensions with discrete levels (0/1 or 0/1/2), each scored separately, with evidence quoted from the output. Never propose a naive 1–10 judge.

Walk the user through what the rubric dimensions would be for their agent. E.g. for a support bot: `correct_action` (static), `policy_followed` (static), `tone_appropriate` (LLM-judge), `addresses_concern` (LLM-judge). See `references/scoring.md`.

### Decision 3: Result vs trace

What you're explaining: "Two ways to look at a run. **Result-only** scores what the agent produced at the end. **Trace evaluation** also scores the steps it took to get there. Result-only is simpler and how most public benchmarks (SWE-bench, OSWorld) work. Trace eval costs more but tells you *where* a failure happened — useful if you said in Phase 1 that you want to know which step went wrong."

Recommend based on the user's Phase 1 answer to "what would you want the harness to tell you?" If they said "where it went wrong", trace eval is worth it. If they said "just whether it worked", start with result-only and add trace later if needed.

These are not exclusive — recommend "both" only if the user genuinely needs step-level diagnosis. See `references/result-vs-trace.md`.

### Decision 4: Dataset construction

What you're explaining: "We'll start with 5–20 hand-crafted cases. Each case is a starting state + an input + what should happen. We'll write them together, targeting the failure modes you named earlier. Over time the dataset grows from real production traffic, support tickets, or synthetic generation — but day one is hand-crafted because that's where you have the most signal."

Help the user draft 3–5 starter case sketches right now, before scaffolding — these become real cases in the scaffold. Each should target a failure mode they named in Phase 1. See `references/eval-cases.md`.

### End of Phase 2

Write a short design doc (markdown, ~1 page) in the user's project at `eval/DESIGN.md` summarizing the four decisions and the starter case sketches. Show it to the user, get confirmation. **This is the source of truth for Phase 3.** If the user wants to change something, change the doc first, then build from it.

---

## Phase 3 — Scaffold

Now build the harness from scratch, tailored to the user's agent and the decisions captured in `eval/DESIGN.md`. Ask the user where to put the harness (default: `eval/` in their project root) and what language (default: Python — pick what fits their stack).

**Write the harness yourself based on the design doc.** Do not lean on a pre-canned template. The whole point of Phase 1–2 was to get the shape right for *this* agent — the file names, types, env adapter, scorers, and case schema should reflect the user's domain, not generic placeholders. Naming matters: a customer-support harness should not have a file called `example_static.py`; it should be `verified_identity_before_refund.py`.

Every scaffold should have these pieces, but their internals are agent-specific:

- **An env adapter** — a small module exposing `reset(case) -> handle` and `teardown(handle)`. The body depends on Decision 1 (Docker container? in-process fakes? SQL fixture? stateless?). Include any supporting artifacts (Dockerfile, fixture SQL, recorded HTTP cassettes) the choice requires.
- **A case dataset** — JSONL or YAML in a `tasks/` directory. 3–5 starter cases, each derived from a specific failure mode named in Phase 1. Each case has a stable ID, an initial-state spec, an input, and an expected outcome (`must_call`, `must_not_call`, a rubric, or whatever fits the agent).
- **Scorers, one file per dimension** — static scorers return a structured record with `passed`, `value`, and `details` (diffs, missing fields). LLM-judge scorers, if any, must use rubric prompts with discrete levels and evidence quotes — never a naive 1–10. Per-dimension breakdowns are stored, never collapsed into one number at scoring time.
- **A runner** — the case loop: for each case, `reset` the env, call the user's agent through a thin adapter, capture result and trace, run every scorer, append a JSONL record to `results/<agent-version>/<timestamp>.jsonl`. Stream a per-case PASS/FAIL line to stdout so the human watching can follow along.
- **A trace schema and step-level scorers** — only if Decision 3 said trace eval. Keep the schema flat and serializable (`index`, `kind`, `payload`, `timestamp`).
- **An agent seam** — one file the user fills in, e.g. `run_agent.py`, that calls into their actual agent and returns `(result, trace)`. Leave a clear TODO and an example shape.
- **A short `README.md`** — written for the user (new to evals): how to add a case, how to add a scorer, how to run the suite. Tie everything back to the language used in Phase 2 so it lands.

Show the user the file tree before generating, get a thumbs-up, then generate. After generating, run the harness end-to-end with a stub agent (returning canned outputs) so the user sees the loop working and the JSONL records appearing.

---

## Phase 4 — Validate and grow

Once the scaffold runs:

1. Have the user wire up `run_agent.py` to their actual agent.
2. Run the 3–5 starter cases. Walk through the output with the user — per case, per dimension. Don't just show a number.
3. For each case where the score "feels wrong" to the user, fix the **scorer or rubric**, not the agent. The scorers are part of what we're calibrating, especially on day one.
4. Discuss enrichment paths (real traffic replay, support tickets, synthetic). Pick one that matches where their agent is in its lifecycle. See `references/eval-cases.md`.
5. Set expectations: the harness will be wrong in places, and that's normal. Cases will get retired, scorers will get refined, the rubric will tighten. The harness is a living artifact.

---

## Operating principles for this skill

- **Discovery is non-negotiable.** If the user says "just build me an eval harness", explain briefly why you need to ask a few questions first, and do Phase 1. A wrong harness wastes more time than the interview saves.
- **Static beats LLM-judge when both work.** If the output is a checkable artifact, do not propose LLM-as-judge for it.
- **Rubrics, always, for LLM-as-judge.** Never call a naive "rate this 1–10" judge.
- **Cases are data, not code.** Keep them in JSONL/YAML so the user can hand-edit and review in PRs.
- **Optimize for diagnosis, not for a leaderboard number.** A harness whose only output is "73%" violates Principle 2.
- **Teach as you build.** Every concept the user doesn't know yet is a chance to explain it briefly and ground it in their agent. By Phase 4 they should be able to add a scorer themselves.

---

## Reference files

Load these on demand when working through the relevant phase — don't read them all upfront, and don't dump them on the user. They exist so you (the assistant) can pull depth when a specific decision is on the table.

- `references/principles.md` — the two first principles, in depth
- `references/simulation-env.md` — env design patterns and reset strategies
- `references/scoring.md` — static scorers, LLM-judge with rubrics, hybrids
- `references/result-vs-trace.md` — when to evaluate the trace
- `references/eval-cases.md` — building and growing the dataset

## A note on no templates

This skill deliberately ships **no code templates**. The harness should be written from scratch each time, shaped by the design doc the user co-authored in Phase 2. Generic templates encourage copy-paste with placeholder names; this skill is opinionated that the file names, types, and scorers should speak the user's domain. The references above are the substrate — write idiomatic code from them, not from a skeleton.
