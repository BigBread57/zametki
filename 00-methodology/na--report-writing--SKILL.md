# Report writing

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / .claude/skills/report-writing/SKILL.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/.claude/skills/report-writing/SKILL.md)

---

---
name: report-writing
description: Use once a challenge is solved (flag verified) to produce the writeup, or when the user asks for a writeup/report on a past or in-progress investigation.
---

# Report writing

Template: [na--knowledge--writeup-template.md](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/na--knowledge--writeup-template.md).

## Discipline

1. **Write from `notes.md`/`artifacts/`, not from memory.** If a challenge
   workspace exists (`challenges/<event>/<slug>/`), the investigation log is
   the source of truth — pulling from conversation memory risks reconstructing
   steps in the wrong order or skipping a dead end that's actually useful
   context for the next similar challenge.
2. **Flag must be verified before writing "Solved: yes."** A flag-shaped
   string that hasn't been confirmed against the challenge's format/scoreboard
   doesn't get written up as solved — mark it as a leading hypothesis instead
   if verification wasn't possible.
3. **Include the actual exploit/script**, not a description of one. If it
   lives in `exploit/solve.py` in the workspace, reference or inline it
   directly rather than re-describing what it does in prose.
4. **Naming and location**: save as
   `writeups/published/<event>-<challenge-slug>.md` at the project root.
5. **What worked during the actual investigation matters more than a clean
   retelling** — if three approaches were tried before the one that worked,
   a one-line note on why the first two failed is more useful to future-you
   than omitting them for a tidier narrative.

## Knowledge base updates — separate from the writeup

If the challenge surfaced a genuinely new, reusable technique or gotcha,
propose it as its own diff to [na--knowledge--vuln-catalog.md](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/na--knowledge--vuln-catalog.md) or the relevant
`knowledge/tools-reference/<category>.md` file — present it for confirmation,
don't fold it silently into the writeup or apply it automatically. This keeps
the knowledge base curated rather than growing unbounded.
