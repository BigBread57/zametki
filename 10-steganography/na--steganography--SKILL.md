# Steganography

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / .claude/skills/steganography/SKILL.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/.claude/skills/steganography/SKILL.md)

---

---
name: steganography
description: Playbook for steganography challenges — an image/audio/video file with a description implying something is hidden inside it (not just "recover a corrupted file", that's forensics). Use after ctf-methodology has classified the challenge as steganography.
---

# Steganography

Reference commands: [na--knowledge--tools-reference--steganography.md](https://raw.githubusercontent.com/BigBread57/zametki/main/10-steganography/na--knowledge--tools-reference--steganography.md).

## Approach

Work through this list for the actual file format — steganography challenges
are almost always solvable by exhausting a short, known technique list per
format, not by cleverness.

1. **Any file, first**: `tools/triage_file.sh <file> <workspace-dir>` should
   already have run per `ctf-methodology` step 1a — re-read its output before
   doing anything else. It's already `file`+`exiftool`+`binwalk`+`strings` in
   one pass, and has directly caught a flag sitting in an `exiftool` warning
   field before (`Trailer data after ... IEND chunk` on a sample whose
   Comment field was a deliberate red herring). If it hasn't run yet for this
   file, run it now. Follow up with `binwalk -e` specifically if the triage
   output hinted at an appended/embedded second file.
2. **PNG/BMP**: LSB steganography is the default assumption.
   `zsteg` is the standard PNG/BMP LSB tool, `pngcheck -v` for structural
   anomalies, `stegoveritas` as a broader second pass.
3. **JPEG**: `steghide extract -sf <file>` first with no passphrase, then
   `-p ""` explicitly, then `stegseek <file> wordlists/Passwords/rockyou.txt`
   to crack the passphrase directly rather than guessing common CTF ones.
4. **WAV/audio**: check a spectrogram — messages hidden in the frequency
   domain show up visually as text/shapes when plotted.
5. **Unknown/ambiguous format**: default to the "any file" step and `strings`
   before guessing at a specific embedding scheme.

## Discipline

Try the passphrase-less/default case for every tool before assuming a
password is required — a large fraction of CTF stego challenges use the
simplest configuration of whichever tool they're built around.


---

## Зависимости и ссылки вне библиотеки

Следующие файлы или ресурсы упомянуты в исходнике, но не входят в эту библиотеку документов. Их наличие и работоспособность здесь не проверялись.

- `tools/triage_file.sh`
