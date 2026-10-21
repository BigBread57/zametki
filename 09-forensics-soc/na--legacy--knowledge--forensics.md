# forensics

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.
> Архивная методика старой реализации; упомянутые функции и обёртки в библиотеку не включены.

Источники этого текста:

- [nonoleekr/CTF-Agent / archive/legacy-python-agent/knowledge/forensics.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/archive/legacy-python-agent/knowledge/forensics.md)

---

Forensics heuristics:

- Start with file type and magic bytes, not the extension — a renamed
  file is a common trick, and `forensics_analyze` already checks the real
  signature rather than trusting the filename.
- Check the filename itself for embedded content, not just the file's
  bytes — flags are sometimes hidden in metadata like the name, not the
  payload.
- Look for a container format (ZIP, PDF) hiding another file inside it, or
  extra bytes appended after a file's normal end-of-data marker.
- For real forensics tooling beyond what's automated here (EXIF metadata,
  embedded-file carving, deleted-file recovery), reach for ExifTool,
  binwalk, or foremost manually — none of these are wired in yet.
- Cross-reference multiple artifacts from the same challenge (a file plus
  its metadata, or several files in a bundle) rather than analyzing each
  in isolation — the flag is often the correlation, not any single file.
