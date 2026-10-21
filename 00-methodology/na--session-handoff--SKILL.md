# Session handoff

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / .claude/skills/session-handoff/SKILL.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/.claude/skills/session-handoff/SKILL.md)

---

---
name: session-handoff
description: Use when a challenge investigation is at risk of losing continuity — approaching a usage/context limit, the user mentions switching to Gemini/OpenCode/ChatGPT, a session is about to end, or you're resuming a workspace that shows signs of being worked on by a different tool. Also use proactively at natural checkpoints (a flag just found, a long investigation about to start) so a recent handoff bundle always exists.
---

# Session handoff

Claude Code cannot detect its own usage limit before hitting it — there's no
tool for that. What this skill covers is making sure a limit or outage never
costs real progress, by keeping `tools/handoff_bundle.sh` current and by
resuming cleanly from one someone else generated.

## When to proactively checkpoint

Run `tools/handoff_bundle.sh <workspace-dir>` (regenerates `handoff.md`,
cheap, safe to run repeatedly) at natural pause points without waiting to be
asked:

- Right after a flag is found and verified, before writing the full writeup.
- Before starting a long, expensive step (a slow brute force, an extended
  symbolic-execution run) — if the session ends mid-step, the bundle should
  reflect the plan and findings up to that point, not nothing.
- If the conversation shows signs of running long (many tool calls, several
  challenges worked in one session) — a stale bundle from an hour ago is much
  less useful than a fresh one.
- If the user says anything about switching tools, running low, or asks you
  to "save progress" / "checkpoint."

Mention that you did it — a one-line note ("checkpointed to
`challenges/.../handoff.md`") is enough, not a production of the bundle's
full contents into the conversation.

## Resuming from a handoff bundle

If a workspace has a `handoff.md` that's newer than the last `notes.md`
entry, or the user hands you one directly, that means a different tool (or a
previous Claude Code session) was working on this and switching back. Read
it fully before doing anything else:

- Don't re-run recon it already logged — check the artifact tails and
  evidence manifest first.
- Continue from its "What to do next" section rather than re-deriving a plan
  from `challenge.md` alone.
- If it references an exploit script draft, read `exploit/` directly — the
  bundle only includes a tail/snapshot, the real file is the source of truth.
- Append your own continuation to `notes.md` as normal — don't treat the
  handoff bundle itself as the living log, it's a snapshot, not a diary.

## What NOT to do

- Don't wait for an explicit "checkpoint now" instruction before the first
  proactive save on a long investigation — by the time it's asked for, it may
  be too late.
- Don't regenerate a bundle so often it becomes noise — tie it to real
  milestones (flag found, expensive step starting, session feels long), not
  every single tool call.
- Don't treat this as a reason to skip the normal `notes.md`/`artifacts/`
  discipline — the handoff bundle is built *from* those files; if they're not
  being kept current, the bundle will be thin regardless of when it's run.


---

## Зависимости и ссылки вне библиотеки

Следующие файлы или ресурсы упомянуты в исходнике, но не входят в эту библиотеку документов. Их наличие и работоспособность здесь не проверялись.

- `tools/handoff_bundle.sh`
