# dsh-trae-agent-team

> Dispatch-and-acceptance discipline: DSH acts as the team lead, TRAE (or DSH's native Agent Teams) runs the members, and the maker never verifies its own work.
> Single-file skill. No runtime, no dependencies — drop \`SKILL.md\` into a skills directory.

**English** ｜ [中文](README.zh.md)

## What it is

An operating discipline that makes multi-agent work **accountable and auditable** — not a framework, not a plugin, no services to run. Ten rules:

1. **Budget first, meeting second** — 2 members (max 4), 3 rounds each, 3 rework rounds, 30 min ceiling; stop and report when exceeded;
2. **Three roles** — lead / maker / checker, one sentence of persona each;
3. **Six-field task contract** — acceptance definition, write scope, dependencies, irreversibility flag, QA checklist (≤12), evidence slot;
4. **Three state-machine invariants** — one `in_progress` per member, no claiming while awaiting acceptance, only the owner mutates state;
5. **maker ≠ checker** — the author never verifies; use a fresh-context subagent, ideally a different model;
6. **One next step per round**, never ten at once;
7. **Cost and blast-radius gates** — rounds, subagent count and parallelism are hard numbers;
8. **Line-item ledger** — time, dispatch, artifacts, duration, rework count;
9. **Shutdown checklist** — artifact paths + criteria re-run + ledger closed;
10. **Cross-runtime orchestration (optional)** — DSH ⇄ TRAE over a task board, a mailbox and two-way wake-ups (plain Node scripts).

## Install

```bash
# DSH / Claude-Code style skills root
mkdir -p ~/.dsh/skills/dsh-trae-agent-team
curl -fsSL https://raw.githubusercontent.com/Myng-i/dsh-trae-agent-team/main/SKILL.md \
  -o ~/.dsh/skills/dsh-trae-agent-team/SKILL.md

# Trae CN / compatible root
mkdir -p ~/.trae-cn/skills/dsh-trae-agent-team
cp ~/.dsh/skills/dsh-trae-agent-team/SKILL.md ~/.trae-cn/skills/dsh-trae-agent-team/SKILL.md
```

Restart the host so it rescans the skills root. **Never keep same-named backups inside a skills root** — the loader resolves them by name; one measured case silently loaded v0.3.0 of another skill instead of v0.3.7.

## Two ways to run it

| | DSH native Agent Teams | Cross-runtime (DSH ⇄ TRAE) |
|---|---|---|
| Interface | `spawn_teammate` / `team_task_*` / `send_message` | `bus-cli.mjs` task board + `mailbox.mjs` + `wake-*.mjs` |
| Best for | multiple roles inside one DSH session | using TRAE's free subagents as members |
| Depends on | nothing (built in) | companion repo [dsh-trae-channel](https://github.com/Myng-i/dsh-trae-channel) |

Dispatch template: [`templates/task-body-template.md`](templates/task-body-template.md) — 11 items plus the title line, 12 sections total; a missing section means the task is not ready.

## Parallelism: measured, not assumed

All verdicts come from filesystem timestamps (subagent-reported times drifted 2–3.6 s in measurement, so they are not evidence):

| Shape | Result |
|---|---|
| 3 separate tasks posted at once | **serial** — zero overlap (a session handles one turn at a time) |
| one task asking for 3 concurrent subagents | true parallel, 4 s overlap |
| 4 subagents | 7.211 s overlap; #4 started 0.69–0.88 s late |
| 5 subagents | 14.761 s overlap; 8.073 s start spread, mostly self-inflicted sandbox retries (1.232 s once told not to retry) |
| 6 subagents | #6 hard-fails: `concurrent subagent limit reached (limit: 5)` — 0 s, no queueing |

⇒ **Parallelism requires "one task, many subagents", not "many tasks". The limit of 5 is hard: drop routes, don't wait.**

## Why the cost rules exist

A single instruction ("reply with one sentence, dispatch nothing") produced **57 model requests, ~730k input tokens and 6 awakened members** under an ungated configuration: the team charter propagates itself and outranks human instructions. Hence hard numbers for budget, parallelism and rework — and stop-and-report instead of silently continuing.

## License

MIT — see [LICENSE](LICENSE).
