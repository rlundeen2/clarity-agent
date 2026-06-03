# Eval framework migration: clarity-agent → RAMPART

## Goal (one line)

Stop maintaining the in-tree eval harness. Run **new** evals on
[RAMPART](https://github.com/microsoft/RAMPART) through a thin adapter, freeze
the legacy cases in place, and delete the old framework once they've drained.

## Why

`evals/framework/` is ~260 KB of bespoke harness (target wrapping, simulated
user, judge, runner, resume cache, summary reporter) with no upstream
maintainer. RAMPART covers the same conceptual ground — simulated user + judge
+ pytest plugin + cache + report — and has an active dev team. The overlap is
structural, not literal. Clarity should let RAMPART carry the maintenance load.

## Strategy: freeze + drain

- The 21 existing cases under `evals/cases/` stay **frozen** on the current
  framework until they're rewritten on RAMPART or age out.
- All **new** cases go in `evals/cases_rampart/` against RAMPART. No new code
  lands in `evals/framework/` after Phase 1.
- When `evals/cases/` is empty (or below the archive threshold), the legacy
  framework, conftest pieces, and CI plumbing are deleted in one pass.

This avoids a flag-day rewrite and avoids porting cases just to delete them
later. The cost is a temporary side-by-side period — and **double eval CI
cost** while both jobs run.

## How this maps to PRs

Two tracks instead of a strict 1→5 sequence. **Track A (parity)** is
decision-driven and gates reliable RAMPART authoring. **Track B (containment)**
is independent and should start immediately. Each row is one PR with an
explicit done-criterion and the test that proves it.

### Phase 1 — Adapter foundation ✅ shipped (PR #55)

- `[evals]` extras pinning `rampart==0.1.0` (strict; bumps are deliberate).
- `evals/rampart_adapter/` — `ClarityAgentAdapter`, `ClarityAgentSession`,
  `AppManifest`, config wrapper.
- `evals/cases_rampart/` — three smoke tests proving the wiring.
- `.github/workflows/evals.yml` — parallel `evals-rampart` job.
- `evals/README.md` — frozen banner.
- Legacy conftest hooks (`pytest_runtest_protocol`,
  `pytest_runtest_makereport`) scoped to `evals/cases/` only.

**Adapter design.** `ClarityAgentSession` bridges RAMPART's async `Session` to
clarity-agent's sync `ClaritySession.chat()` via `asyncio.to_thread`. It owns
priming (system prompt with behaviors + process guide + packet-status report).
Per turn it observes tool calls (`backend.on_tool_call` → `Response.tool_calls`,
per-turn closures restored in `finally`), side effects (`.clarity-protocol/`
mtime-snapshot diff → `Response.side_effects`), and cost (`backend.on_cost`,
per-turn and cumulative in `Response.metadata`). `__aexit__` tears down the
inner session, disconnects the backend, then closes the transcript — each step
best-effort.

### Track A — Parity (decision-driven; finish A1–A2 before broad authoring)

| PR  | Feature | Disposition | Done-criterion |
| --- | --- | --- | --- |
| A1  | `STATUS: ONGOING/CLOSING/DONE` termination + goodbye-detection | (a) re-implement | Custom `PromptDriver`/`LLMDriver` subclass; a `cases_rampart/` test proves a conversation **ends** on a DONE signal and **doesn't** silently run to `max_turns`. |
| A2  | `refusal_acceptable` success short-circuit | (a) re-implement, or (b) drop with rationale | Custom evaluator + outcome resolver; a test proves an early refusal resolves as success, not a false failure. RAMPART has no native short-circuit — if it can't be reproduced cleanly, drop it and record the decision. |
| A3  | Regression-sorting calibration | new | Run a small set of **known-pass** and **known-fail** fixtures through RAMPART; confirm it sorts them correctly. NOT a verdict-for-verdict diff against the legacy judge — the outcome models differ. |
| A4  | Precondition gates (persona-adoption turn 1, substantivity, goal-pursued) | (a) re-implement | Precondition `Evaluator`s; one test each. |
| A5  | Observability profile correctness | (a) fix | `observability_profile` is **derived from the actual backend**, or the adapter asserts the target isn't the SDK backend. Closes the seam where the adapter advertises `TOOL_AND_SIDE_EFFECTS` but the SDK backend's Bash-invoked actions never fire `on_tool_call`. |
| A6  | `advisory` decorator | (a) keep | Framework-agnostic pytest mark; verify it still routes outcomes under the RAMPART path. |
| A7  | Reporting | (c) decide | Keep `summary.md` with `.clarity-protocol/` deep links, or accept RAMPART's report. Resolve the open question rather than carrying it. |

Dropped outright (record as decisions, no PR beyond removal):
`--rebuild=conversation|judgment|all` and the SHA-256 transcript-fingerprint
judge cache. **Before dropping**, confirm RAMPART's cache invalidates on
transcript change at comparable fidelity — the legacy cache's cheap
single-criterion iteration is a real workflow, not just plumbing.

### Track B — Containment (independent; start now)

| PR  | Item | Done-criterion |
| --- | --- | --- |
| B1  | Freeze-enforcement CI | `tests/test_evals_frozen.py` in the **normal** pytest suite (not the eval workflow) fails if a new `.py` appears under `evals/cases/` past a baseline. Whitelist-based so renames/refactors of existing files pass. Lands early so the banner is actually enforced during coexistence. |
| B2  | Docs + drain tracker | `UPDATE-CHECKLIST.md` gains "new cases → `evals/cases_rampart/`". A **checklist issue** lists each frozen case + its disposition — this is what draining PRs cross-link to, and it makes progress visible per-PR. (Supersedes "track a count in README".) |
| B3  | Delete (terminal) | When `evals/cases/` hits the archive threshold: delete `evals/framework/`, strip framework code from `evals/conftest.py`, delete the B1 frozen-check, collapse `evals/cases_rampart/` → `evals/cases/`, simplify `evals.yml`, drop the legacy README section. |

**Drain is a policy, not a phase.** When a legacy case needs a behavior
change, the contributor rewrites it on RAMPART and deletes the old version
rather than editing in place. No standalone scheduled work.

## Open caveats carried forward

- **Tool observation under the SDK backend.** Bash-invoked `ai_actions` don't
  fire `on_tool_call`, so `Response.tool_calls` can be empty even when an action
  ran. Today evals point `target` at `anthropic` (native tool-use) so this is
  latent, but A5 makes it explicit. Evaluators needing cross-backend tool
  observation should also inspect `Response.side_effects`.
- **Cost cap.** No per-session/per-test ceiling yet. Use RAMPART's if it grows
  one; otherwise a wrapper that aborts a session over budget.
- **Concurrency.** `asyncio.to_thread` lets two RAMPART tasks each hold a
  session, but they share process-global state (env, cwd). `-n` worker-level
  parallelism (xdist) is safe; in-process asyncio parallelism that mutates env
  vars is not.

## Risks

- **RAMPART API churn (pre-1.0).** We pin and bump deliberately. Add an owner:
  upstream bumps land via their own PR with the `cases_rampart` smoke suite as
  the gate.
- **Async bridge.** `ClaritySession` is sync with stateful per-turn context.
  Run the smoke suite with `-n 2` to flush blocking-loop surprises early.
- **`refusal_acceptable` semantics.** Covered by A2 — reproduce or drop.

## Non-goals

- No feature-parity sweep. A frozen case that relies on a behavior that doesn't
  survive Track A stays frozen or is dropped.
- No required upstream contributions. The migration completes either way.
- No new eval capability as part of this work. Priority is stability.

## Open questions

- Where does `AppManifest` live long-term — `evals/rampart_adapter/` or nearer
  `src/clarity_agent/` for reuse? Defer until a second consumer appears.
