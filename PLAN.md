# pi-prove-it — Plan

A port of [prove_it](https://github.com/searlsco/prove_it) (by Justin Searls) to the
[Pi coding agent](https://github.com/earendil-works/pi). Same idea: config-driven
verification tasks that force the agent's code to actually work. MIT licensed.

## Core concept

A Pi extension (TypeScript, loaded natively via jiti — no build step) plus a
companion CLI (`prove-it`). Reads `.pi/prove-it/config.json` in a project and runs
matching tasks when Pi fires lifecycle events. Tasks are `script` (exit-code
pass/fail), `agent` (AI reviewer), or `env` (variable injection).

## Event mapping (Claude Code → Pi)

| prove_it event | Pi event | Behavior |
|---|---|---|
| SessionStart | `session_start` | Briefing, env tasks, git baseline, session state init |
| PreToolUse | `tool_call` | Guards: block via `{ block: true, reason }` |
| PostToolUse / PostToolUseFailure | `tool_result` | File-edit tracking, test-run detection, TDD state machine |
| Stop | `agent_before_settle` / `turn_end` | Prove-done: agent review; FAIL → `{ continue: true }` with feedback |
| pre-commit / pre-push | CLI shim (`prove-it hook git:<event>`) | Same dispatcher semantics |

Key Pi adaptations:

- **Tool names are lowercase** (`read`, `bash`, `edit`, `write`) — matchers adapt.
- **Agent reviewers shell out to `pi -p --no-session`** (default) instead of
  `claude -p`; `gpt-*` models route to `codex exec -` for adversarial review.
- **Stop blocking** uses the `agent_before_settle` continuation contract
  (`{ continue: true, entries: [...] }`) with a consecutive-failure loop guard.
- **Env tasks** feed `taskEnv` for subprocesses (Pi has no CLAUDE_ENV_FILE).
- **Session state** in `~/.pi-prove-it/sessions/` keyed by Pi session id.
- **Config**: `.pi/prove-it/config.json` (project) +
  `~/.pi/agent/prove-it/config.json` (global), cascade with `config.local.json`,
  hooks merged by task name. Defaults live in one module, no `\|\|` fallbacks.

## How we prove it's working (3 verification tiers)

1. **In-process extension harness** (the workhorse): a mock `ExtensionAPI` +
   `ExtensionContext` that records registrations and lets tests fire synthetic
   events (`tool_call`, `agent_before_settle`, …) and assert blocking reasons,
   continuation requests, and briefing content. Analogue of prove_it's
   `hook-harness.js`. Every feature's correctness is pinned here.
2. **Unit tests** (`node:test`, zero deps, `standard` lint): config cascade,
   template expansion, `when` conditions, verdict parsing, git churn refs
   (real temp git repos), session state store.
3. **E2E against real Pi**: fixture projects + `pi --extension . -p "<prompt>"
   --mode json`; assert transcript shows briefing injected, bash blocked by a
   guard, and stop held until the test script passes. API-key-gated (skips
   cleanly without one), model overridable via `PI_PROVE_IT_E2E_MODEL`.

`prove-it doctor` also self-checks an installation (pi on PATH, config parse,
reviewer command reachable).

## Work breakdown

Tracked in beads (`bd list`). Epics:

1. **Scaffold + verification foundation** — repo, harness, unit suite, e2e runner
2. **Config layer** — defaults, cascade, hooks merge, guard-config
3. **Session lifecycle** — skeleton/wiring, state store, briefing + env tasks
4. **Guards + tool blocking** — tool_call guards, script runner, TDD machine
5. **Stop flow + agent reviews** — reviewer runner, prove-done loop, async/parallel/arbiter
6. **Conditions + templates** — `when` evaluation, `{{var}}` expansion
7. **CLI + git hooks** — install/init/doctor/monitor, git shim dispatcher
8. **Packaging + docs** — pi package manifest, reviewer skills, examples, README

Order of attack: scaffold → harness → config → session state → guards →
templates → stop flow → CLI/hooks → skills/docs/examples.
