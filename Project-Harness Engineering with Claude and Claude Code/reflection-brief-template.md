# Reflection Brief — Harness Engineering Capstone

**Name:** Rithvik Rajani Kanth Sasi
**Date:** 2026-11-09

Everything I cite is in [`evidence/`](./evidence/), one folder per system.

**Environment**

- Model(s): `claude-haiku-4-5-20251001` and `claude-sonnet-4-6` (System 1 second run, via `--model`), live through the Vocareum proxy at `https://claude.vocareum.com`; System 4 ran offline against `fixtures/recorded_responses/shift_C_2026-04-30.json`; System 3 needs no model.
- OS / Python: Windows 11 Pro, Python 3.12.10, a separate `.venv` per system (`anthropic` 0.39.0 / 0.69.0 / 2.6.0).
- Tests: all four suites pass — 30, 30, 35 and 33, plus the System 3 validator printing `OK`. Logs are in each system's evidence folder ([System 1](./evidence/system-1-claims-intake/pytest.log), [System 2](./evidence/system-2-retail-context/pytest.log), [System 3](./evidence/system-3-claude-code-config/pytest-and-validator.log), [System 4](./evidence/system-4-shift-monitor/pytest.log)).
- Approx. API spend: ~$1.15. System 1 reports $0.1086 and $0.1025 for two Haiku runs and $0.5570 for Sonnet. System 2 prints no cost; from [`budget.json`](./evidence/system-2-retail-context/artifacts/budget.json) plus the 8 eval calls it's roughly $0.19 a run, run twice. System 4 was free.

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → [`artifacts-sonnet/traces/claim_03_water_damage.jsonl`](./evidence/system-1-claims-intake/artifacts-sonnet/traces/claim_03_water_damage.jsonl) goes `tool_use, tool_use, tool_use, tool_use, tool_use, tool_use, end_turn` across 7 turns. The decision is in `run()` in `claims_intake/loop.py`: it returns a `FinalState` on `end_turn`, appends the tool results and continues on `tool_use`, and raises `UnexpectedStopReason` on anything else. There's no counter in the `while` at all — it's `while True`, and the only other way out is `budget.check()` raising `BudgetExceeded`, so the model really does own the control flow.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → `test_no_string_membership_against_text_in_loop` fails the build if `loop.py` contains any `"token" in <something>` expression, i.e. control flow by searching the assistant's text. My run shows why that matters: in [`artifacts-haiku/traces/claim_01_kitchen_fire.jsonl`](./evidence/system-1-claims-intake/artifacts-haiku/traces/claim_01_kitchen_fire.jsonl) the last turn is `end_turn` with no tool call, and the text was a question about repair costs. A keyword check would have found nothing to match and either spun again or fired on the wrong token, whereas dispatching on `stop_reason` ended the turn correctly and let the backstop take it.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → `route_to_adjuster(queue, claim_summary)` and `escalate_to_human(reason, structured_summary)` overlap — both terminal, both carrying a summary of the same claim. Both descriptions open with "TERMINAL TOOL." and separate on a number rather than tone: route at confidence of at least 0.6 with severity assessed, escalate below 0.6. So the model checks the `confidence` it already passed to `classify_claim`. The structured error that earns its keep is the overlap guard — both terminal dispatchers check `session.terminal_called` before anything else, so whichever fires second gets back `{"is_error": true, "error_category": "permanent", "is_retryable": false, "message": "terminal tool already called this claim"}`. The `is_retryable: false` is the part a prose string can't really carry. `test_tools.py` covers both guards, inside the 30 passing tests in [`pytest.log`](./evidence/system-1-claims-intake/pytest.log).

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → `claim_03_water_damage` on Sonnet: 7 turns, 1 clarification, 26994 input / 1345 output tokens, $0.1012 ([`artifacts-sonnet/summary.md`](./evidence/system-1-claims-intake/artifacts-sonnet/summary.md)). On Haiku the same claim took 5 turns and about $0.02, and only 2 turns on the re-run. The README expects roughly $0.05 for all 8 with 7 routed and 1 escalated; my Haiku runs came to $0.1086 and $0.1025 at 2 routed / 6 escalated, and Sonnet $0.5570 at 5 routed / 3 escalated. The cost gap is mostly that every turn resends the system prompt and all seven tool schemas over a growing transcript, and Sonnet costs more per token. The outcome gap is Q19.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → Run `20261109-183347` ([`budget.json`](./evidence/system-2-retail-context/artifacts/budget.json)): baseline 38708, assembled 16931, 56.26% reduction. `active` dominates at 15789 tokens, 93% of what's left. It stays byte-exact because the payment-method issue is still open — the AVS_MISMATCH code, card last-4 7782, and which retry failed in what order are all still load-bearing, and you can't know which turn the next decision hangs on. The count came from the real `messages.count_tokens` endpoint, which `budget.json` records as model-authoritative.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → Resolved threads get summarized, the one active thread stays verbatim. `per_section_tokens` in [`budget.json`](./evidence/system-2-retail-context/artifacts/budget.json) shows it: `resolved_refund` 355 and `resolved_subscription` 601, against `active` at 15789 untouched. `compression_api` shows what that cost — the refund thread went in at 12344 tokens and came back 342, the subscription 11485 in and 588 out. So roughly 24k tokens of settled conversation became under 1k, and the live thread wasn't touched at all.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → All six pass against the full context ([`eval.jsonl`](./evidence/system-2-retail-context/artifacts/eval.jsonl), 6/6). In the stripped control Q1 still answers $22.14 for ORD-77310, but Q6 fails ([`eval_control.jsonl`](./evidence/system-2-retail-context/artifacts/eval_control.jsonl)) — it asks for the exact status token and the answer opens "# Unknown", saying none is recorded. That's the facts block earning its 204 tokens: `in_progress` is a token the block states outright and the raw conversation never does, so 15789 tokens of verbatim transcript couldn't recover it.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → From [`.claude/rules/tests.md`](./evidence/system-3-claude-code-config/config-snapshot/.claude/rules/tests.md):

   ```yaml
   paths:
     - "**/*.test.tsx"
     - "**/*.test.ts"
   ```

   Tests here are co-located — `CLAUDE.md` says `Foo.tsx` lives next to `Foo.test.tsx`. A directory-level `CLAUDE.md` would need copying into `src/components/`, `src/pages/` and `src/api/`, and would still fire on the non-test files sitting there. The glob follows the naming instead of the folder, so one file covers the tree and activates only on test files.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → From [`.claude/skills/deploy-check/SKILL.md`](./evidence/system-3-claude-code-config/config-snapshot/.claude/skills/deploy-check/SKILL.md): `context: fork`, then `allowed-tools:` listing `Read`, `Grep`, `Glob` and a handful of read-only shell entries — `Bash(git status:*)`, `Bash(git diff:*)` and the rest of the `git`/`gh pr` reads. Forking keeps that git and gh output in a sub-agent and returns one pass/fail summary, so a pre-deploy check doesn't eat the window you were working in. The allowlist has no `Write`, `Edit` or bare `Bash`, so a skill called deploy-check structurally cannot deploy or change anything. Without the fork the transcript floods your session; without the allowlist you're trusting prose to restrain a tool that has a shell.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → `python -m ecommerce_team_config .` prints `OK` and exits 0 ([`pytest-and-validator.log`](./evidence/system-3-claude-code-config/pytest-and-validator.log)), and what it validates is the project-level tree: `./CLAUDE.md`, the four `@`-imported `.claude/standards/`, the three `.claude/rules/`, the `/review` command and the `deploy-check` skill. Project example: `.claude/rules/tests.md`, in git and loaded for every teammate. User example: `~/.claude/CLAUDE.md` and `~/.claude/commands/`, which the scope table calls "Not shared via version control" and illustrates with a personal `/morning` command. The troubleshooting note is the practical version — run `/memory`, because a convention that isn't being followed is usually sitting user-level where nobody else gets it.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → The warm tier holds 40 rows after seeding from `fixtures/defects.json`; `defects_since('2026-04-01T00:00:00Z', limit=50)` returned 17 ([`run-shift.log`](./evidence/system-4-shift-monitor/run-shift.log)). The query is `WarmStore.defects_since` in `shift_monitor/warm.py` — `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?` — and `EXPLAIN` confirms it isn't scanning: `SEARCH defects USING INDEX idx_defects_ts (ts>?)`. The model never sees the full history because SQLite filters before anything is serialized into a prompt, so prompt size tracks the shift window, not the age of the database.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → `recovery.decide()` returns `fresh` if the manifest has no steps or is already complete, `resume` if the last step is within `STALE_RESUME_THRESHOLD_MINUTES = 30`, and `fresh` otherwise. I walked the boundary in [`recovery-decisions.log`](./evidence/system-4-shift-monitor/recovery-decisions.log): 29 and 30 minutes both `resume`, 31 and 45 `fresh`, and an empty or complete manifest `fresh` regardless of age. The docstring explains the number: 30 minutes is roughly 1/16 of an 8-hour shift, so a resume inside that window is still working the same shift's working set. Past it the world has moved — new defects, changed alerts — and replaying a half-finished conversation carries assumptions that were true half an hour ago. A fresh start with the manifest's findings injected gives a small, internally consistent context instead of a stale one.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → Mine is 980 bytes ([`hot_state.json`](./evidence/system-4-shift-monitor/data-after-run/hot_state.json)), against `HOT_STATE_BYTE_BUDGET = 5_120` in `shift_monitor/state.py`. The budget is enforced rather than advisory — `write_atomic` raises before writing if the payload is over, and `recent_defect_hashes` is capped at `MAX_RECENT_HASHES = 20` by the Pydantic field. It matters because this file enters the prompt every shift forever, and anything that only grows raises the cost of each invocation until it stops fitting. The cap forces the question of what's worth carrying, which is why the state holds hashes and alert strings rather than defect records.

---

## Part 2 — Synthesis

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: `claims_intake/system_prompt.py`. The four claim types, the severity bands and the 0.6 threshold exist only as prose, and `test_no_claim_type_equality_branching_in_package` fails the suite if that judgement leaks into Python.
    → Harness: `claims_intake/loop.py`, `run()`. It owns the `stop_reason` dispatch, the `Budget` and the `Tracer` that wrote [`claim_03_water_damage.jsonl`](./evidence/system-1-claims-intake/artifacts-sonnet/traces/claim_03_water_damage.jsonl), and knows nothing about insurance.
    → Orchestration: `shift_monitor/pipeline.py` `run_shift`, with `manifest.py`, `recovery.py` and `fork.py`. It runs once per shift and hands the next invocation 980 bytes of hot state plus a SQLite file, which is what makes "runs forever" possible.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → Guaranteed in code: `_escalate_without_terminal` in `claims_intake/run.py`. Every claim ends routed or escalated whether the model cooperates or not, and it fired six times in my Haiku run — [`escalations.jsonl`](./evidence/system-1-claims-intake/artifacts-haiku/escalations.jsonl) is six lines all reading "model ended its turn without a terminal action (deterministic backstop)". Guided by prompt: the 0.6 rule and the instruction to ask one clarifying question via `request_clarification`, which is exactly what didn't hold. System 4 enforces the same way from the other end — `state.py` `write_atomic` raises before writing if hot state exceeds `HOT_STATE_BYTE_BUDGET = 5_120`, and mine landed at 980 bytes ([`hot_state.json`](./evidence/system-4-shift-monitor/data-after-run/hot_state.json)). Put the invariant in code when you can't afford to lose it and it doesn't depend on the facts; leave it to the prompt when the right answer depends on what was just learned.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 works inside one conversation: [`budget.json`](./evidence/system-2-retail-context/artifacts/budget.json) shows 38708 baseline tokens down to 16931, the live thread kept whole at 15789 and the two settled threads crushed to under a thousand combined. System 4 works between conversations: 980 bytes carried forward, 17 of 40 warm rows pulled for the window, the other 23 left in `warm.sqlite` unseen. The shared principle is that only decision-load-bearing content occupies the window. The mechanisms are near opposites — System 2 has one long history it must keep, so it compresses and reorders it; System 4 never accumulates a history and re-queries an indexed store each time.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → `tests/test_run_backstop.py` scripts a fake client returning `end_turn` with no tool call and asserts the result is still `outcome == "escalated"` with a line in `escalations.jsonl`. A clean run never shows that, because on a good day the model calls a terminal tool. `test_no_integer_literal_iteration_cap_in_loop` is the same category: a run finishing in 7 turns looks identical whether the loop exits on `stop_reason` or a hidden `while turn < 10`, and the difference only surfaces on the claim needing 11. System 4's `tests/test_us03_crash_recovery.py` is the same idea one layer up: a shift that never crashes exercises none of the resume-vs-fresh logic, which is why I had to drive it by hand in [`recovery-decisions.log`](./evidence/system-4-shift-monitor/recovery-decisions.log). All of it is behaviour you only see on the bad path.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → System 1. Every tool with side effects writes inside `runs/<timestamp>/` and nowhere else, so nothing reaches a real adjuster system and the worst case is a claim in the wrong file — which is what happened to `claim_01_kitchen_fire`, escalated where the fixture expects routed ([`escalations.jsonl`](./evidence/system-1-claims-intake/artifacts-haiku/escalations.jsonl)). Enforcement is the dispatcher short-circuiting on `session.terminal_called` so a second terminal call is a graceful error rather than a double-route, plus `Budget(max_input_tokens, max_wall_clock_s)` raising `BudgetExceeded` instead of truncating. The kill switch is `--max-input-tokens` / `--max-wall-clock-s`, and each run being a fresh timestamped directory you can delete. The queue write is the part I'd harden before this went near production.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → Two things. System 2 exited 1 on `UnicodeEncodeError: 'charmap' codec can't encode character '≥'` — the final `print` in `run.py` contains a `≥` and the Windows console is cp1252. Every artifact was already written, so it was cosmetic; `PYTHONIOENCODING=utf-8` gave exit 0 ([`run-all.log`](./evidence/system-2-retail-context/run-all.log)). The bigger one was System 1 returning 2 routed / 6 escalated against the README's 7 and 1, every escalation from the backstop rather than an `escalate_to_human` call. I wrapped the client in a spy printing `response.model` and per-turn text, and on `claim_01` the model asked "Before I estimate the damage, do you have a rough idea of the cost to repair..." as prose and ended its turn instead of calling `request_clarification` — and that tool is the claimant's only input channel, so nothing could answer. A second Haiku run escalated the same six claims ([`artifacts-haiku-run2/summary.md`](./evidence/system-1-claims-intake/artifacts-haiku-run2/summary.md)) with only turn counts moving, and `claude-sonnet-4-6` on the same code got 5 routed / 3 escalated — so it's reproducible, and the loop and tools are fine; the prompt just doesn't pin a newer model to the tool.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I'd stop letting the backstop be the only answer to a missing terminal call. A model slip and a real judgement call currently produce the same artifact, and in both Haiku runs six claims were logged as "escalated" with an empty `claim_type` and `severity: -` in [`summary.md`](./evidence/system-1-claims-intake/artifacts-haiku/summary.md) — an escalation queue where most entries aren't decisions. On `end_turn` with `session.terminal_called` still false, I'd append one corrective turn saying a terminal tool is required and continue, escalating only if it stops again. That keeps `stop_reason` as the control signal and costs one extra turn, but the queue goes back to meaning "a human should look at this".
