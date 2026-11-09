# Evidence

Logs and run artifacts from my own machine, one folder per system. The numbers and the reasoning are in the [reflection brief](../reflection-brief-template.md) — this file is just a map of what's where.

Windows 11 Pro, Python 3.12.10, a separate `.venv` per system. Systems 1 and 2 hit the live API through the Vocareum proxy, System 4 ran offline with `--recorded-response`, System 3 needs no model.

| Folder | Tests | Run artifacts |
|---|---|---|
| `system-1-claims-intake/` | 30 passed | three runs, see below |
| `system-2-retail-context/` | 30 passed | `artifacts/` — budget, assembled context, eval + control |
| `system-3-claude-code-config/` | 35 passed, validator `OK` | `config-snapshot/` — the hierarchy the validator checks |
| `system-4-shift-monitor/` | 33 passed | `data-after-run/` — hot state, scratchpad, both forks |

## Why System 1 has three runs

The first run didn't match the 7-routed/1-escalated split the exercise README predicts, so I ran it twice more to work out why before writing it up:

- `artifacts-haiku/` — Haiku 4.5, 2 routed / 6 escalated, $0.1086
- `artifacts-haiku-run2/` — Haiku 4.5 again, the same six claims escalated, $0.1025 (so it's reproducible, not flaky)
- `artifacts-sonnet/` — Sonnet 4.6 via `--model`, 5 routed / 3 escalated, $0.5570

Q19 of the brief has the cause. Each folder keeps its `summary.md`, `escalations.jsonl` and all eight `traces/`; the Sonnet run also has the `queues/` claims routed into. For a trace that shows the loop doing what the exercise is about, read `artifacts-sonnet/traces/claim_03_water_damage.jsonl` — seven turns with a clarification in the middle.

## Notes on two logs

`system-2-retail-context/run-all.log` is the clean re-run. The first attempt wrote every artifact and then exited 1 on a `UnicodeEncodeError` printing `≥` to a cp1252 console.

`system-4-shift-monitor/run-shift.log` holds two shift runs — the first on the default `--since` (8 hours ago, correctly finding nothing, since the fixtures are dated March–April 2026) and the second on an explicit window.
