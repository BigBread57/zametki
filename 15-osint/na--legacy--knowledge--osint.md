# osint

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.
> Архивная методика старой реализации; упомянутые функции и обёртки в библиотеку не включены.

Источники этого текста:

- [nonoleekr/CTF-Agent / archive/legacy-python-agent/knowledge/osint.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/archive/legacy-python-agent/knowledge/osint.md)

---

OSINT heuristics:

- Extract usernames, handles, email addresses, domains, and IPs from any source text first.
- Correlate handles across social platforms and public code hosting before assuming the data is noise.
- Treat domain names, profile links, and archive snapshots as investigative pivots.
- Look for leaked metadata, paste sites, and cached pages when the challenge mentions a person or organization.
- Prefer entity mapping over brute force: identify people, aliases, and shared infrastructure.
