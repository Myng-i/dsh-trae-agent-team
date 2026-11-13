# dsh-trae-agent-team

DSH leads, TRAE runs the members: a discipline for dispatch, acceptance and bookkeeping. Single-file skill, no runtime dependencies.

**English** ｜ [中文](README.zh.md)

## Ten rules

1. Budget first: 2 members (max 4), 3 rounds each, 3 rework rounds, 30 min ceiling — stop and report when exceeded;
2. Three roles: lead dispatches, maker executes, checker verifies;
3. Six-field task contract: acceptance definition, write scope, dependencies, irreversibility flag, QA checklist, evidence slot;
4. State machine: one `in_progress` per member, no claiming while awaiting acceptance, only the owner mutates state;
5. maker ≠ checker: the author never verifies; hand acceptance to a fresh-context subagent;
6. One next step per round;
7. Gates: rounds, subagent count and parallelism are hard numbers;
8. Line-item ledger: time, dispatch, artifacts, duration, rework;
9. Shutdown: artifact paths, criteria re-run, ledger closed;
10. Cross-runtime: task board, mailbox, two-way wake-ups (optional).

With no gates, a single instruction — reply with one sentence, dispatch nothing — produced 57 model requests, ~730k input tokens and 6 awakened members. The numbers above are ceilings, not suggestions.

## Install

```bash
mkdir -p ~/.dsh/skills/dsh-trae-agent-team
curl -fsSL https://raw.githubusercontent.com/Myng-i/dsh-trae-agent-team/main/SKILL.md \
  -o ~/.dsh/skills/dsh-trae-agent-team/SKILL.md

mkdir -p ~/.trae-cn/skills/dsh-trae-agent-team
cp ~/.dsh/skills/dsh-trae-agent-team/SKILL.md ~/.trae-cn/skills/dsh-trae-agent-team/SKILL.md
```

Restart the host afterwards. Keep backups outside the skills root — a same-named backup is loaded as the real skill.

## Usage

| | DSH native Agent Teams | Cross-runtime (DSH ⇄ TRAE) |
|---|---|---|
| Interface | `spawn_teammate` / `team_task_*` / `send_message` | `bus-cli.mjs` task board + `mailbox.mjs` + `wake-*.mjs` |
| Best for | multiple roles in one DSH session | TRAE's free subagents as members |
| Depends on | nothing | companion repo [dsh-trae-channel](https://github.com/Myng-i/dsh-trae-channel) |

Dispatch template: [`templates/task-body-template.md`](templates/task-body-template.md) — twelve sections, all required.

## Parallelism (measured)

Verdicts use filesystem timestamps; subagent-reported times drift 2–3.6 s and are not evidence.

| Shape | Result |
|---|---|
| 3 tasks posted at once | serial, zero overlap |
| 3 subagents in one task | true parallel, 4 s overlap |
| 4 routes | 7.211 s overlap |
| 5 routes | 14.761 s overlap |
| 6 routes | #6 fails: `concurrent subagent limit reached (limit: 5)`, no queueing |

⇒ Parallelism means one task with many subagents, not many tasks. The limit is 5; drop routes instead of waiting.

## License

MIT, see [LICENSE](LICENSE).
