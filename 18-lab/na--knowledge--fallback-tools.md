# Fallback tools — when Claude Code isn't available

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / knowledge/fallback-tools.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/knowledge/fallback-tools.md)

---

# Fallback tools — when Claude Code isn't available

Claude Code is the primary agent for this project (see `CLAUDE.md`). Everything
below is for the specific situation where it's unavailable mid-competition —
usage limit hit, an outage, or you need a second opinion without spending
Claude Code budget. **This is a manual human process** — nothing in this repo
can automatically detect "Claude Code is out of tokens" and switch tools for
you; you'll see that message yourself and need to act on it. What this file
and `tools/handoff_bundle.sh` do is make that switch fast and lossless instead
of starting over.

## The four tools, what each is actually for

| Tool | Role | Access |
|---|---|---|
| **Claude Code** | Primary solver — full agent loop, all skills/knowledge/MCP servers in this repo | This session |
| **Gemini** | Secondary solver — analysis, debugging a stuck approach, documentation lookup, a second opinion on reasoning you're not confident in | `GEMINI_API_KEY` in `.env`, see below |
| **OpenCode** | Fallback *agent environment* — a real terminal coding agent (not just chat), already installed, can be pointed at Gemini or other backends | Already on PATH, see below |
| **ChatGPT (free web)** | Manual-only backup — no CLI, no file access, pure copy/paste | chat.openai.com in a browser |

Gemini and OpenCode aren't equivalent to each other: Gemini (via the CLI) is
a single-shot or short-session reasoning tool — good for "look at this and
tell me what you think." OpenCode is a full agentic loop with its own tool
use, closer to what Claude Code itself does — good for "keep actually solving
this" when Claude Code is the thing that's down.

## Gemini — verified working setup

```
cd /home/ronald/CTF-Agent
set -a; source .env; set +a
npx -y @google/gemini-cli --skip-trust -p "<your prompt>"
```

- `--skip-trust` is required in headless/non-interactive use — without it the
  CLI refuses to run outside an interactively-trusted directory.
- The API key is already in `.env` as `GEMINI_API_KEY`.

**Known limitation, hit directly while setting this up**: the free-tier key
in `.env` has a hard request quota (`generativelanguage.googleapis.com/
generate_content_free_tier_requests, limit: 20`) — the CLI retries on a 429
for about a minute before surfacing a clear quota-exceeded error. If Gemini
suddenly stops responding, this is the most likely reason, not a bug. If
you're relying on Gemini as a real fallback during a competition, either
attach billing to that key beforehand or treat quota exhaustion as a reason
to fall through to OpenCode/ChatGPT next, not something to debug mid-round.

**Large prompts are fine content-wise but count against the same quota faster**
(more tokens ≈ more likely to trip the per-request or per-minute limit) — the
handoff bundle (see below) is intentionally trimmed (artifact tails, not full
logs) to keep individual asks small.

## OpenCode — verified working setup

Already installed (`~/.opencode/bin`, on `PATH` via `.bashrc`), already
authenticated for Google's provider using the same key:

```
cd /home/ronald/CTF-Agent
set -a; source .env; set +a
opencode run "<your prompt>" --model google/gemini-3-flash-preview
```

**The one real gotcha**: OpenCode's Google provider reads
`GOOGLE_GENERATIVE_AI_API_KEY`, not `GEMINI_API_KEY` — same value, different
variable name. `.env` already has both (kept in sync manually; if you rotate
the Gemini key, update both lines). Without the second variable you get
`Google Generative AI API key is missing` even though `GEMINI_API_KEY` is set.

For interactive use instead of one-shot `run`, just `opencode` in the project
directory — it's a full TUI agent, similar shape to Claude Code itself
(reads files, runs shell commands, etc.), so an in-progress investigation can
genuinely continue there, not just get commented on.

OpenRouter is also configured in `.env` (`OPENROUTER_API_KEY`) as a second
backend option for OpenCode, but wasn't fully verified working during setup —
`opencode auth login openrouter` hit a `fetch() URL is invalid` error in this
environment, and passing the key via env var alone produced a generic server
error. If Gemini's quota is exhausted and OpenRouter is the next thing to try,
expect to troubleshoot that login flow first — don't assume it's ready.

## ChatGPT (free web) — manual only

No API key, no CLI access from this environment. Purely: open
chat.openai.com, paste the handoff bundle (below) as your first message,
work in the browser, and manually copy anything worth keeping back into
`notes.md`/`exploit/` when you're done or switch back. Treat it as the last
resort when Gemini's quota is gone and OpenCode isn't cooperating either.

## The actual switching procedure

1. **You'll see Claude Code's own usage-limit message** — that's the
   notification; nothing here needs to detect it separately.
2. **Generate a handoff bundle**: `tools/triage_file.sh` and challenge work
   should already be saving to the workspace as you go (per `CLAUDE.md`), so
   run `tools/handoff_bundle.sh <workspace-dir>` to compile
   `challenge.md` + `notes.md` + evidence manifest + recent artifact tails +
   any exploit script into one `handoff.md` file, ready to paste.
3. **Open the fallback tool** (Gemini CLI, OpenCode, or ChatGPT web) and paste
   `handoff.md`'s contents as the first message.
4. **Keep working there** — but write anything worth keeping back into the
   *same workspace files* (`notes.md`, `exploit/`) as you go, by hand if the
   tool can't touch the filesystem itself (Gemini one-shot, ChatGPT). This is
   what makes switching back to Claude Code later lossless — it just reads
   the same files, no bundle needed to resume there.
5. **When Claude Code is available again**, just continue the conversation —
   the workspace files carry everything forward automatically.

## Why this works at all

This project already keeps investigation state in files (`challenge.md`,
`notes.md`, `artifacts/`, `evidence/`, `exploit/`) rather than only in
conversation context — see `knowledge/environment.md`'s workspace convention.
That discipline is what makes tool-switching practical: a different LLM with
zero shared history can still pick up real progress because the progress
never lived only in one tool's memory to begin with.


---

## Зависимости и ссылки вне библиотеки

Следующие файлы или ресурсы упомянуты в исходнике, но не входят в эту библиотеку документов. Их наличие и работоспособность здесь не проверялись.

- `tools/handoff_bundle.sh`
- `tools/triage_file.sh`
