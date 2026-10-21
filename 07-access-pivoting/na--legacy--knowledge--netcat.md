# netcat

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.
> Архивная методика старой реализации; упомянутые функции и обёртки в библиотеку не включены.

Источники этого текста:

- [nonoleekr/CTF-Agent / archive/legacy-python-agent/knowledge/netcat.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/archive/legacy-python-agent/knowledge/netcat.md)

---

Netcat heuristics:

- Look for a host and port stated directly in the challenge text —
  `nc_analyze` extracts these from the common `connect to HOST at port
  PORT`, `nc HOST PORT`, and `HOST:PORT` phrasings automatically.
- A connection is only ever attempted if `CTF_AGENT_ALLOW_REMOTE_NC` is
  explicitly set — by default the tool only reports the host/port it
  found, which is the expected, safe behavior, not a failure.
- Just connect and read first: a surprising number of these challenges
  send the flag the moment you connect, with no input required at all —
  exactly what `nc_analyze` does.
- `nc_analyze` never sends data, only reads — a challenge that expects
  you to type something back (a menu, a prompt, a protocol handshake)
  needs a human to take over from there; this is a known, deliberate
  limitation, not a bug.
- Don't assume a service is HTTP or another known protocol just because
  it's reachable over TCP — read what it actually sends before guessing
  at the protocol.
