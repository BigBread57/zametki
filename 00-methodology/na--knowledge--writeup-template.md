# Writeup template

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / knowledge/writeup-template.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/knowledge/writeup-template.md)

---

# Writeup template

Populate this from `notes.md` and `artifacts/` in the challenge workspace —
don't reconstruct from memory after the fact. Write it immediately after the
flag is verified, while detail is fresh. Save as `writeups/published/<event>-<slug>.md`.

```markdown
# <Challenge Name> — <Category>

**Event:** <ctf name>  **Points:** <n>  **Solved:** yes/no

## Summary

One or two sentences: what the vulnerability/technique was and how it was found.

## Provided

- Description: <verbatim or summarized>
- Files: <list, with sha256 if binaries>
- Target: <host:port if remote>

## Investigation

Chronological, terse — pull straight from notes.md:

1. <tool run> → <what it showed>
2. <hypothesis formed> → <how it was tested>
3. ...

## Root cause / technique

What the actual bug or weakness was, precisely enough that someone could
reproduce it without re-deriving your reasoning.

## Exploit

```
<the actual working script or command sequence, from exploits/ if promoted there>
```

## Flag

`flag{...}` — confirmed against: <challenge-stated format / server response / scoreboard accept>

## Notes for next time

Only include this section if something genuinely reusable surfaced — a gotcha,
a technique worth adding to `knowledge/vuln-catalog.md` or
`knowledge/tools-reference/<category>.md`. Propose that addition as a separate
diff, don't fold it silently into the writeup.
```
