# ssh

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.
> Архивная методика старой реализации; упомянутые функции и обёртки в библиотеку не включены.

Источники этого текста:

- [nonoleekr/CTF-Agent / archive/legacy-python-agent/knowledge/ssh.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/archive/legacy-python-agent/knowledge/ssh.md)

---

SSH heuristics:

- Look for a host, port, username, and password stated directly in the
  challenge text — `ssh_analyze` extracts these from the two common
  picoCTF phrasings (`ssh as USER to HOST at port PORT` and
  `USER@HOST -p PORT`) automatically; anything else needs a manual read.
- A connection is only ever attempted if `CTF_AGENT_ALLOW_REMOTE_SSH` is
  explicitly set — by default the tool only reports what it found in the
  text, which is the expected, safe behavior, not a failure.
- Once connected, check the login banner/MOTD before running anything
  else — some challenge servers print the flag (or a strong hint) the
  moment you log in, before any command even runs.
- `ssh_analyze` only runs one small, fixed, read-only recon command
  (`whoami`/`pwd`/`ls`/`cat`/`find` for `flag*`). A harder multi-step
  challenge (following a hint into another directory, checking `sudo -l`,
  reading a config file) needs a human to take over from there.
- Never assume a password found in one part of the text applies to a
  different host/user mentioned elsewhere — only trust a
  host/username/password triple that's clearly associated together.
