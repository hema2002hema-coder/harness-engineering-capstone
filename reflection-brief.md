# Reflection Brief — Harness Engineering Capstone

**Name:** Desuru Hemalatha
**Date:** 2026-10-09

**Environment**

- Model(s): claude-haiku-4-5-20251001 (Systems 1, 4), claude-sonnet-4-6 (System 1 comparison run), retail_context default model (System 2)
- OS / Python: Linux, Python 3.13.0
- Approx. API spend: ~$0.60 total across all runs

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** From `claim_05_auto_collision.jsonl`: turn 1 `stop_reason=tool_use` (lookup_policy + 6 record_claim_fact calls), turn 2 `tool_use` (classify_claim, assess_severity), turn 3 `tool_use` (route_to_adjuster), turn 4 `end_turn`. The decision lives in `claims_intake/loop.py`, function `run()`: the `while True` loop returns when `response.stop_reason == "end_turn"`, continues when `== "tool_use"`, and raises `UnexpectedStopReason` for anything else.

2. **Anti-pattern.** `test_antipatterns.py::test_no_integer_literal_iteration_cap_in_loop` statically checks that `loop.py` never uses a fixed-count `for`/`while` as its stopping condition. If it did, claims needing more turns (claim_04/claim_08 took 5 turns, claim_03 took 7 turns with a clarification in different runs) would get truncated before reaching `route_to_adjuster`, silently corrupting the outcome.

3. **Tool design.** `record_claim_fact` (field/value pairs) and `classify_claim` (claim_type/confidence/rationale) have non-overlapping schemas, so the model can't confuse recording a fact with making a classification. A structured JSON tool error (`is_error=true`, per the `ToolExecutor` contract in `loop.py`) lets the agent adjust its next call programmatically; a generic string error gives it nothing to branch on.

4. **Your numbers.** Run `20260927_053506`, claim_05_auto_collision: 4 turns, 14,827 input / 912 output tokens, est. $0.0194. Three of eight claims in this run (claim_01, claim_02, claim_06) terminated `incomplete` — the model returned `end_turn` without calling a terminal tool. Reproducible across both Haiku and Sonnet reruns (claim_01 and claim_07 failed in every attempt), showing this is a real model behavior, not a harness bug.

### System 2 — Context strategy

5. **The reduction.** From `budget.json` (run `20260927-055346`): baseline 38,708 tokens → assembled 16,903 tokens = 56.33% reduction. The `active` section dominates at 15,789 tokens (93% of assembled) because it's the unresolved, in-progress issue that still needs full fidelity to answer follow-up questions correctly.

6. **Summarize vs preserve.** Resolved segments are summarized: `resolved_refund` compressed from 12,334 input tokens to 430 output tokens, `resolved_subscription` from 11,475 to 498. The `active` segment (15,789 tokens) stays byte-exact because it's still open and a follow-up question may need an exact value from it. `case_facts` (204 tokens) is structurally extracted into a fixed 12-field record, neither summarized nor verbatim.

7. **Facts block.** Comparing `eval.jsonl` to `eval_control.jsonl`: Q6 ("the exact status token from the case record") passed on the full context but failed as expected on the control — proving the case-facts block carries structured, exact-value information the free-text transcript doesn't reliably preserve. Q1 passed on both since the refund amount was independently recoverable from the active transcript.

### System 3 — Claude Code config

8. **Path-scoped rules.** From `.claude/rules/react.md` frontmatter: `paths: ["src/components/**/*", "src/pages/**/*"]`. Better than a directory-level CLAUDE.md because components/pages aren't confined to one directory in this monorepo — the rule loads only when a matching file is open, keeping unrelated sessions free of React-specific noise.

9. **Forked skill.** From `.claude/skills/deploy-check/SKILL.md`: `context: fork` with `allowed-tools` limited to `Read, Grep, Glob` and read-only `Bash(git status:*)`, `Bash(git diff:*)`, `Bash(git log:*)`, `Bash(git rev-parse:*)`, `Bash(git ls-files:*)`, `Bash(gh pr view:*)`, `Bash(gh pr checks:*)`. Forking isolates the check's exploration from the main session; read-only enforcement guarantees a "check" skill can't accidentally mutate the repo it's validating.

10. **Scope.** From `CLAUDE.md`: project-level scope is `./CLAUDE.md`, `.claude/standards/`, `.claude/rules/` — lives in git, shared with the team (e.g. `.claude/rules/react.md`). User-level scope is `~/.claude/CLAUDE.md`, `~/.claude/commands/`, `~/.claude/skills/` — personal, not version-controlled.

### System 4 — Orchestration

11. **Push work down.** The seeding script inserts all fixture defects into `data/warm.sqlite`, but the shift run queries only defects `since` a recent cutoff — in my run, `since=2026-09-26T21:12:10Z` returned `new=0` because fixture data predates that window, showing the SQL filter (not the model) decides what's "new."

12. **Crash recovery.** From `recovery.py`: `decide()` returns `"resume"` only if the manifest has steps, isn't complete, and the last step is within `STALE_RESUME_THRESHOLD_MINUTES = 30`; otherwise `"fresh"`. Beyond 30 minutes, a fresh start with an injected summary is more reliable because real-world shift conditions may have moved on, and resuming a stale partial risks reasoning from an outdated snapshot.

13. **Small state.** My `data/hot_state.json` was **643 bytes** after a run — well under the ~5KB budget. This matters because the system runs once per shift indefinitely; an unbounded file would grow every shift and become expensive to read/write and include in every future prompt.

---

## Part 2 — Synthesis

14. **Three layers.**
    → Model: `claims_intake`'s routing/escalation decisions — the model, not Python, decides `claim_type`/severity via tool calls (enforced by `test_no_claim_type_equality_branching_in_package`).
    → Harness: `.claude/skills/deploy-check/SKILL.md` — deterministic tool allowlisting and forked context are harness-level guarantees.
    → Orchestration: `recovery.py`'s `decide()` function — tiered state and the 30-minute staleness threshold are orchestration-layer state management, independent of any single model call.

15. **Deterministic vs prompt.** Deterministic: `.claude/skills/deploy-check/SKILL.md`'s `allowed-tools` is a hard allowlist restricting Bash to specific read-only subcommands, enforced by the tool-calling layer. Prompt-guided: `claims_intake`'s escalation judgment for low-confidence claims varied across identical reruns (claim_06 sometimes resolved `incomplete`, sometimes properly `escalated`), showing it's guided, not guaranteed. Deterministic enforcement suits catastrophic-violation risks; prompt guidance suits genuinely context-dependent judgment calls.

16. **Context, two faces.** System 2 manages context *within* one long conversation: `budget.json` shows 38,708 raw tokens compressed to 16,903 assembled (56.33% reduction) via summarizing resolved segments. System 4 manages context *across* sessions: `hot_state.json` (643 bytes) persists only a tiny working set between shifts, with history pushed into `warm.sqlite` and pulled back via filtered SQL. Same principle — don't carry more than the current task needs — different mechanism.

17. **Reliability you can't see in one run.** `test_no_integer_literal_iteration_cap_in_loop` statically guarantees no hidden turn cap exists in `loop.py` — a single successful run, even 8/8 clean, could still be passing "by luck" under a generous cap without ever revealing the cap exists, until a claim needing more turns silently truncates in production.

18. **Blast radius.** System 3's `/deploy-check` skill: if it misbehaved, blast radius is limited to a wrong pass/fail summary — it structurally cannot mutate the repo since `allowed-tools` excludes any writing Bash command. The kill switch is the allowlist itself: removing a tool from `allowed-tools` is a one-line, git-tracked change that immediately revokes that capability.

---

## Part 3 — Honest assessment

19. **What broke.** My Anthropic API calls initially failed with `401 invalid x-api-key` because I was using a Vocareum-proxy key (`voc-...`) against `api.anthropic.com` directly. Fixed by also setting `ANTHROPIC_BASE_URL=https://claude.vocareum.com` so requests routed through the correct proxy.

20. **What you'd change.** I'd add an explicit "must call a terminal tool before accepting end_turn" check to System 1's loop. Even though `stop_reason`-driven control is the intended design, my runs showed the model can reach `end_turn` without calling `route_to_adjuster`/escalation, producing `incomplete` outcomes on 3-5 of 8 claims across multiple runs and models. A lightweight post-hoc assertion (not a turn cap) would catch this without violating the anti-pattern the test suite enforces.
