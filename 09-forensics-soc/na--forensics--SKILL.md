# Digital forensics

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / .claude/skills/forensics/SKILL.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/.claude/skills/forensics/SKILL.md)

---

---
name: forensics
description: Playbook for digital forensics challenges — a disk/memory image, network capture (pcap), archive/container file, or a description asking you to recover/find something inside a provided artifact. Use after ctf-methodology has classified the challenge as forensics.
---

# Digital forensics

Reference commands: [na--knowledge--tools-reference--forensics.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/na--knowledge--tools-reference--forensics.md).

## Approach

1. **Triage first, unconditionally.** `tools/triage_file.sh <file>
   <workspace-dir>` should already have run per `ctf-methodology` step 1a —
   it covers metadata (`exiftool`), file identification (`file`), and an
   embedded-signature check (`binwalk`) in one pass. GPS data, author names,
   software versions, and occasionally the flag itself show up here directly.
   Re-read its output before anything else; run it now if it hasn't run yet.
2. **File/archive/container**: if the triage output hinted at an embedded
   file, `binwalk -e` to extract → re-run classification on whatever comes
   out (forensics challenges are often several layers deep).
3. **Memory image**: `.venv/bin/vol -f <image> windows.info` (or
   `linux.banners`) first to identify the profile, then pick plugins based on
   what you're after — `pslist`/`cmdline` for process activity,
   `linux.bash`/`windows.cmdline` for command history.
4. **Network capture**: `tshark -r <cap> -q -z conv,tcp` for a fast overview,
   then `--export-objects http,out/` to pull transferred files directly —
   that's the fastest path to "there's a file hidden in this pcap," try it
   before manually following individual streams.
5. **Disk image with no obvious file layer**: `foremost`/`scalpel` as a
   broader carving sweep once targeted extraction (`binwalk -e`) has come up
   empty.

## Discipline

Forensics challenges are usually solvable by working through this list for
the artifact type you actually have, not by cleverness — resist the urge to
skip straight to an advanced technique before the cheap, obvious steps have
been exhausted.


---

## Зависимости и ссылки вне библиотеки

Следующие файлы или ресурсы упомянуты в исходнике, но не входят в эту библиотеку документов. Их наличие и работоспособность здесь не проверялись.

- `tools/triage_file.sh`
