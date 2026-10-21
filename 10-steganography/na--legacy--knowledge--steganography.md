# steganography

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.
> Архивная методика старой реализации; упомянутые функции и обёртки в библиотеку не включены.

Источники этого текста:

- [nonoleekr/CTF-Agent / archive/legacy-python-agent/knowledge/steganography.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/archive/legacy-python-agent/knowledge/steganography.md)

---

Steganography heuristics:

- Check for trailing data appended after the image's real end-of-data
  marker first (PNG `IEND`, JPEG EOI) — `stego_analyze` checks this
  automatically, and it's the single most common simple CTF stego trick.
- For PNG specifically, check the ancillary text chunks (`tEXt`, `zTXt`,
  `iTXt`) before assuming the flag is hidden in pixel data — flags are
  frequently placed there directly, and `stego_analyze` extracts them
  natively with no image decoding required.
- Also check for another file's magic bytes embedded anywhere in the
  byte stream (a ZIP, another image, an executable) — `stego_analyze`
  scans for this too.
- If none of the above finds anything, the flag may genuinely be in the
  pixel data (LSB steganography) — that needs real image decoding
  (Pillow, zsteg, Stegsolve) and isn't automated natively here yet.
- Don't skip the mundane checks (file type, metadata) just because the
  challenge says "steganography" — a surprising number of these
  challenges are solved by `file`/`exiftool`-level inspection alone.
