# web

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.
> Архивная методика старой реализации; упомянутые функции и обёртки в библиотеку не включены.

Источники этого текста:

- [nonoleekr/CTF-Agent / archive/legacy-python-agent/knowledge/web.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/archive/legacy-python-agent/knowledge/web.md)

---

Web heuristics:

- Read the request/response carefully before touching anything —
  `web_analyze` parses the method, path, and headers from raw HTTP text,
  and a flag or parameter hint is often sitting in the path or a header
  verbatim.
- A parameter-style path (`?key=value`) is worth specific attention —
  it's flagged by `web_analyze` and is the most common injection surface
  (SQL injection, IDOR via a predictable ID, command injection).
- Check for the obvious authentication/authorization bypasses first:
  default credentials, a predictable session token, or a client-side
  check that a server never re-verifies.
- Cookies, hidden form fields, and comments in the raw HTML are cheap to
  check and frequently contain the actual clue in simpler web challenges.
- Live request modification, fuzzing, and running `sqlmap` against a real
  endpoint are the natural next steps for anything beyond static
  inspection, and aren't automated here yet — this agent doesn't send
  live requests today, only analyzes provided text.
