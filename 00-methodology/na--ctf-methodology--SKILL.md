# CTF methodology

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / .claude/skills/ctf-methodology/SKILL.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/.claude/skills/ctf-methodology/SKILL.md)

---

---
name: ctf-methodology
description: Entry point for any CTF challenge — text description, uploaded files, screenshots, binaries, or a network capture. Use this first, before any category-specific work, to classify the challenge and set up the investigation loop. Triggers on phrases like "solve this challenge", "here's a CTF", "flag{", or any file attachment presented without further context.
---

# CTF methodology

This is the meta-skill. It runs once at the start of every challenge, decides
category, sets up the workspace, and defines the loop that the category skills
(`web-exploitation`, `reverse-engineering`, `binary-exploitation`,
`cryptography`, `forensics`, `steganography`, `osint`, `malware-analysis`)
operate inside of.

## 1. Set up the workspace immediately

Before any analysis, create `challenges/<event>/<slug>/` with `challenge.md`
(paste the description, list provided files, note target host:port if any) and
an empty `notes.md`. Do this even for a challenge that looks trivial — the
workspace is where findings get recorded as you go, not reconstructed
afterward. Full convention: `knowledge/environment.md`.

## 1a. Triage every attached file (including images/screenshots)

Before classifying, run `tools/triage_file.sh <file> <workspace-dir>` on
every file the challenge provides. One command does what would otherwise be
four separate tool calls: copies the file into `evidence/` (deduped by
hash), runs `file` + `exiftool` + `binwalk` + `strings` in one pass, logs the
combined output to `artifacts/`, and records the file in an evidence
manifest. Read its output before deciding on a category in step 2 — it
regularly surfaces the classification signal itself (a magic-byte mismatch,
an `exiftool` warning about trailer data, a flag sitting directly in
`strings` output), not just prep work for later.

## 2. Classify

Apply [na--knowledge--classification-guide.md](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/na--knowledge--classification-guide.md): magic bytes / `file` output first,
extension second, host:port presence third, description keywords last as a
tiebreaker. State which signals you used and what category they point to. If
signals conflict, say so, commit to the strongest one, and name the runner-up —
don't silently discard the disagreement.

Once classified, the matching category skill takes over for tool selection and
technique. This skill's job is done for that part — don't duplicate
category-specific playbooks here.

## 3. The loop

1. **Cheap recon first, always.** File triage already ran in step 1a; add
   `checksec` and other category-specific cheap checks — before anything
   slow, stateful, or destructive.
2. **Log every result to `notes.md` immediately**, with a timestamp. Don't hold
   findings only in conversation context.
3. **Pick the smallest tool that answers the current specific question.** Not
   "run everything" — look up the exact command in
   `knowledge/tools-reference/<category>.md` rather than guessing flags from
   memory.
4. **Re-check for a flag after every step**, not just at the end — a lucky
   early find shouldn't get missed three steps later because you weren't
   looking yet.
5. **Stuck after ~5-6 unproductive steps?** Stop and explicitly reconsider: is
   the classification wrong (go back to step 2), or does this specific step
   need Opus instead of Sonnet (nontrivial ROP chain, multi-stage RSA attack,
   heavily obfuscated reverse engineering)? Don't keep grinding on the same
   approach past that point.
6. **Verify before declaring done.** A flag-shaped string is not a flag until
   it matches the challenge's stated format and, where the challenge allows,
   is actually confirmed (submitted / server accepts it).
7. **Write up immediately** using [na--knowledge--writeup-template.md](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/na--knowledge--writeup-template.md), sourced from
   `notes.md`/`artifacts/` — not reconstructed from memory later, while it's
   fresh.
8. **Propose a knowledge update only if something genuinely new surfaced** — a
   diff to [na--knowledge--vuln-catalog.md](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/na--knowledge--vuln-catalog.md) or a `tools-reference` file, presented
   for confirmation, not auto-applied.

## Hard stops — always ask the user first

- Scanning or sending traffic to any host not explicitly given as the
  challenge target.
- Executing an unknown/untrusted binary directly rather than analyzing it
  statically.
- Anything destructive or hard to reverse in this environment.

These aren't heuristics to weigh against convenience — they're the same "ask
before risky/irreversible actions" standard that applies to all other work,
applied here.


---

## Зависимости и ссылки вне библиотеки

Следующие файлы или ресурсы упомянуты в исходнике, но не входят в эту библиотеку документов. Их наличие и работоспособность здесь не проверялись.

- `tools/triage_file.sh`
