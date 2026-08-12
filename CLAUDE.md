# CLAUDE.md

Project instructions for building AI workflows in this repo.

Three patterns cover nearly everything we build: **Agents**, **Loops**, and
**Graphs**. Use the simplest one that solves the problem. Most bugs in this
codebase come from reaching for a heavier pattern than the task needed.

## Project

<!-- FILL IN -->

- **What this is:**
- **Stack / model provider:**
- **Entry point:**
- **Deeper docs:** see @README.md and @docs/architecture.md

## Commands

<!-- FILL IN — Claude should run these, not guess -->

```bash
# install
# run locally
# test
# lint + typecheck
# eval suite
```

Run lint, typecheck, and the closest test before reporting a task complete.

---

## Core mental model

| Pattern    | Shape                          | Use when                                              |
| ---------- | ------------------------------ | ----------------------------------------------------- |
| **Agent**  | goal → agent → actions         | The goal is clear, the path isn't. Needs tools.        |
| **Loop**   | agent → self → agent           | First answer is unreliable. Needs to check its work.   |
| **Graph**  | agent → workers → one output   | Work splits into independent parts. Needs parallelism. |

These compose. A graph node is usually an agent, and that agent may loop
internally. Compose deliberately — don't let it happen by accident.

### Escalation ladder

Start at the top. Move down only when the current rung demonstrably fails on a
real eval case, and say which case in the PR description.

1. Single prompt
2. Prompt + tools
3. Agent
4. Agent + loop
5. Graph of agents

Each rung adds latency, token cost, and debugging surface. Skipping rungs is
the most common mistake here.

---

## Agents — autonomous, goal-driven

An agent receives a **goal**, decides on **actions**, and executes them.

- One goal per agent, stated in one sentence. If the goal statement needs "and",
  it's two agents or a graph.
- Give each agent the narrowest tool list that can achieve the goal. Unused
  tools in context degrade selection accuracy.
- Every agent needs a hard step budget and an explicit termination condition.
  No agent runs unbounded.
- Tools return structured results, never prose to be re-parsed. Errors come back
  as data the agent can act on, not exceptions that kill the run.
- Agents never mutate shared state directly. They return a result; the caller
  decides what to persist.

## Loops — thinks before it answers

A loop is an agent reconsidering its own output before returning it.

- **Always cap iterations.** Default to 3 unless there's a measured reason to go
  higher. Hitting the cap is a logged event, not a silent fallthrough.
- Every iteration must change state. If iteration N produces the same output as
  N-1, exit — that's convergence, not progress.
- Define the exit criterion up front and make it checkable: a validator passes,
  a schema parses, tests go green, a score clears a threshold. "Looks better" is
  not an exit criterion.
- Prefer an external check (compiler, test run, schema validator) over
  self-assessment. A model grading its own work is the weakest available signal.
- Log each iteration's input, output, and exit decision. A loop you can't replay
  is a loop you can't debug.

## Graphs — many agents, one answer

An orchestrator fans work out to specialized workers and reduces their results
into a single output.

- The orchestrator decomposes and merges. It does not do worker jobs itself.
- Workers are **independent and stateless**. A worker never calls another worker
  — if B needs A's output, that's a sequential step in the orchestrator, not an
  edge between workers.
- Fan out in parallel. Sequential fan-out means the graph was the wrong pattern.
- Exactly one reducer produces the final output. Conflicts between workers are
  resolved by an explicit written rule, not by whichever result arrives last.
- Worker failure is partial, not fatal. Return what succeeded, mark what didn't,
  let the reducer decide whether the result is usable.
- Keep the graph a DAG. No cycles between workers — iteration belongs inside a
  node as a loop.

---

## Conventions

- **Prompts** live in version control as files, not inline string literals.
  They're reviewed like code.
- **State** passed between nodes is a typed, serializable object. If it can't be
  serialized, it can't cross a node boundary.
- **Model choice** is configuration, not a hardcoded constant. Workers doing
  narrow, mechanical jobs should use a smaller model than the orchestrator.
- **Retries** are bounded and use backoff. Distinguish retryable failures (rate
  limit, timeout) from terminal ones (bad input, refusal) and don't retry the
  latter.
- **Never** put credentials, keys, or user PII into a prompt.

## Testing

- Every agent, loop, and graph node gets fixture-based tests with a mocked model
  client. Deterministic tests must not hit the network.
- Maintain an eval set of real cases. Prompt or topology changes require an eval
  run, and the results go in the PR.
- Test the failure paths explicitly: tool error, malformed output, loop hitting
  its cap, worker timing out.

## Observability

Every run emits a trace containing: the goal, each model call with token counts,
each tool invocation with arguments and result, loop iteration count and exit
reason, and total latency and cost.

If you can't answer "why did it do that?" from the trace alone, the
instrumentation is incomplete — fix that before debugging the behavior.

## Budgets

<!-- FILL IN with real numbers -->

- Max tokens per run:
- Max wall-clock latency:
- Max cost per run:

Exceeding a budget aborts the run and logs it. Budgets are enforced in code, not
by convention.

---

## Anti-patterns

- A graph where a single agent would do
- An unbounded loop, or one whose exit depends on the model saying "done"
- An agent handed every available tool "just in case"
- Workers that call each other, creating hidden sequencing
- Retry logic wrapped around a prompt that's wrong rather than flaky
- Prompts edited without an eval run
- Swallowing tool errors so the run appears to succeed

## Notes

- This file is **context, not enforcement**. Anything that must hold — budget
  caps, secret scanning, schema validation — belongs in code, tests, CI, or a
  PreToolUse hook.
- Keep this file short. Link to deeper docs rather than expanding it into a
  manual; long instruction files get followed less consistently.
- Task-specific multi-step workflows belong in `.claude/skills/<name>/SKILL.md`,
  not here.
