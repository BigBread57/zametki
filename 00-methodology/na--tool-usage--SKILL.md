# Tool usage index

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / .claude/skills/tool-usage/SKILL.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/.claude/skills/tool-usage/SKILL.md)

---

---
name: tool-usage
description: Index for "which tool/file do I need for X" when it's not obvious which category skill applies yet, or for live ssh/netcat connection challenges that don't fit the file-based categories. Use when uncertain where to look, or for a bare host:port challenge with no attached file.
---

# Tool usage index

This skill is deliberately thin — it points at maintained references instead
of duplicating them, so there's only one place to update each fact.

## Where things live

| Need | Look in |
|---|---|
| Classify a challenge | [na--knowledge--classification-guide.md](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/na--knowledge--classification-guide.md) |
| Exact command/flags for a category | `knowledge/tools-reference/<category>.md` |
| Fast cross-category one-liner lookup | [na--knowledge--command-cheatsheet.md](https://raw.githubusercontent.com/BigBread57/zametki/main/18-lab/na--knowledge--command-cheatsheet.md) |
| A vulnerability tell → what it means | [na--knowledge--vuln-catalog.md](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/na--knowledge--vuln-catalog.md) |
| A reusable exploit skeleton | [na--knowledge--exploit-patterns.md](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/na--knowledge--exploit-patterns.md) |
| What's installed / where / venv vs PATH | `knowledge/environment.md` |
| Writeup format | [na--knowledge--writeup-template.md](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/na--knowledge--writeup-template.md) |

## Live connection challenges (ssh / netcat) — not file-based, handle here

These don't fit the classification guide's file-signal-first approach since
there's often no file at all — just a description.

1. Extract host/port/credentials directly from the text. Common phrasings:
   `ssh as USER to HOST at port PORT`, `USER@HOST -p PORT`,
   `nc HOST PORT`, `connect to HOST at port PORT`.
2. **Netcat**: connect and just read first — `nc <host> <port>`. Many CTF
   services send the flag immediately on connect, no input required. Don't
   assume you need to send something before checking.
3. **SSH**: connect, check the login banner and home directory
   (`ls`, `pwd`, `cat` on anything flag-shaped) before searching further.
4. Treat this as the "live" case regardless of what the connected service
   turns out to be — if it starts presenting web/crypto/binary content once
   connected, switch to the matching category skill at that point.

## When nothing above resolves it

Fall back to `ctf-methodology`'s classification step and general recon
(`file`, `strings`) — the tool-usage index exists to save a lookup once
category is known, not to replace classification.
