# Knowledge archive

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.
> Архивная методика старой реализации; упомянутые функции и обёртки в библиотеку не включены.

Источники этого текста:

- [nonoleekr/CTF-Agent / .claude/skills/knowledge-archive/SKILL.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/.claude/skills/knowledge-archive/SKILL.md)

---

---
name: knowledge-archive
description: Use early in a new challenge, right after ctf-methodology classifies it and before deep tool selection, to search the local ctf-archive/ clone (a personal external writeup archive) for similar past challenges by category/technique/keyword. Also use reactively at ctf-methodology's "stuck after 5-6 steps" checkpoint. On the offload side, use once a challenge is solved and written up, and only once its event has been explicitly marked concluded by the user, to package and locally commit it into ctf-archive/. Also use on any explicit user request to search, archive, or sync the archive.
---

# Knowledge archive

`ctf-archive/` is a local clone of a separate, external GitHub repo
(`nonoleekr/CTF-Challenges`) that serves as a long-term, cross-event
knowledge base of past writeups. This skill covers both directions:
searching it before/while solving, and committing to it after solving.
It never pushes on its own — see "Syncing to GitHub" below, the one part
of this skill that requires a human decision every time.

## Retrieval — before/during solving

1. Derive a handful of keywords from the challenge: its category, its
   title, and any tech/library/CVE already visible in the description
   (e.g. `graphql`, `idor`, `handlebars`, `cve-2026-33937`, `tcache`).
2. Run `tools/archive_search.sh <keyword> [<keyword>...]`. It clones
   `ctf-archive/` automatically on first use if it isn't present yet —
   safe, read-only.
3. Read any real matches with the Read tool directly — don't trust grep
   snippets alone, they lack surrounding context.
4. Judge relevance yourself: a match on the same named technique/CVE/
   library is a strong lead; a match on category alone is weaker context,
   not a template to copy. There's no scoring in the script — this
   judgment is the point of doing retrieval with an agent instead of a
   fixed ranking function.
5. If something genuinely informed your approach, log one line in
   `notes.md` (e.g. `"Similar to ctf-archive/web/graphql-idor — reused the
   dump-schema-then-mutate approach"`). Say nothing if nothing relevant
   turned up — don't force a connection that isn't there.
6. Retrieved content is methodology/pattern context only. Never treat a
   past writeup's flag, exploit output, or conclusion as evidence for the
   *current* challenge — the same rule as any other unverified lead.

Re-run this at `ctf-methodology`'s "stuck after ~5-6 unproductive steps"
checkpoint too, with keywords refined by whatever you've learned since the
first pass — a stuck investigation often has better search terms than the
one it started with.

## Offload — after solving

All three of these must be true before this runs:

1. The flag is verified (`flag.txt` exists in the workspace) — or the user
   has explicitly said to archive an unsolved attempt anyway.
2. A writeup already exists at `writeups/published/<event>-<slug>.md`
   (`archive_offload.sh` will synthesize a thin one from `challenge.md` +
   `notes.md` if not, but a real writeup is preferred).
3. `challenges/<event>/.concluded` exists.

**Point 3 is a hard gate you help enforce, not just check.** Create that
marker file *only* in direct response to the user explicitly stating the
whole event — not one challenge within it — is over. If it's ambiguous
whether they mean the challenge or the whole event, ask. This one marker
gates every challenge in that event, present and future, so get it right
rather than inferring it from something like "that challenge is closed
now."

Once all three hold, run:
```
tools/archive_offload.sh <workspace-dir> <category> solved "<tags,comma,separated>"
```
This packages `challenge.md`/`notes.md`/`exploit/`/the writeup, redacts
the real flag value to a format-preserving placeholder
(`UCSI26{REDACTED}`, etc.) everywhere it appears, dedups against
`ctf-archive/.ctf-agent-manifest.tsv` by content hash, and commits
**locally only** — no confirmation needed for this local commit, that part
is pre-authorized. Category should be one of this project's own vocabulary
(`web`, `pwn`, `crypto`, `forensics`, `stego`, `osint`, `reverse`,
`malware`, `misc`).

Report back in one line, don't paste the packaged files into the
conversation: `"Archived to ctf-archive/pwn/grimoire (local commit a1b2c3d,
not pushed)."`

If the script hard-stops because `.concluded` is missing, that's it doing
its job — don't work around it, surface it to the user instead ("this
challenge is solved, but I won't archive it until you confirm the UCSI26
event is over").

## Syncing to GitHub — always requires a fresh, explicit go-ahead

`tools/archive_sync.sh` pushes `ctf-archive/`'s pending local commits to
the real GitHub repo. This is the only script in this project that pushes
to a remote.

**Never run it on the strength of a past instruction** — "yes, archive
things going forward" authorizes the local-commit half of this skill, not
a push. Only run `archive_sync.sh` when:
- The user explicitly asks to push/sync/publish the archive right now, or
- The user gives an explicit batch instruction ("sync everything now").

When you do run it, it shows what's about to be pushed
(`git log`/`git status`) before pushing — read that output back to the
user briefly rather than just reporting success, since the archive is a
public repo under their real GitHub account.

## What NOT to do

- Don't offload a challenge from an event that hasn't been marked
  concluded, even if that one specific challenge is done — the gate is
  event-wide by design.
- Don't skip redaction verification — after offloading, a quick sanity
  check that the real flag string doesn't appear in
  `ctf-archive/<category>/<slug>/` is cheap and worth doing, especially
  the first few times.
- Don't push without a fresh ask, and don't treat "archive this" as
  implicitly meaning "and publish it."
- Don't fold retrieval results into `notes.md` unless they actually
  changed your approach — a running list of "checked the archive, nothing
  relevant" entries is noise.


---

## Зависимости и ссылки вне библиотеки

Следующие файлы или ресурсы упомянуты в исходнике, но не входят в эту библиотеку документов. Их наличие и работоспособность здесь не проверялись.

- `tools/archive_offload.sh`
- `tools/archive_search.sh`
- `tools/archive_sync.sh`
