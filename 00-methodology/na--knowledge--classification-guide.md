# Classification guide

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / knowledge/classification-guide.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/knowledge/classification-guide.md)

---

# Classification guide

The old pipeline classified with a first-match keyword ladder (`pwn > reverse >
web > osint > crypto > forensics > steganography > default forensics`) — brittle
by construction: a stego challenge whose description happened to mention "hidden
metadata" matched forensics first and never got reconsidered. Don't reproduce
that. Score multiple signals and stay willing to reclassify if later evidence
disagrees with the first guess.

## Signals, in order of reliability

1. **File magic bytes / `file` output** — the strongest signal when a file is
   provided. Trust this over description text if they conflict.
2. **File extension** — corroborating, not authoritative (extensions lie in
   forensics/stego challenges on purpose).
3. **A host:port or connection string in the description** — usually pwn, web,
   or a live ssh/netcat challenge, not a static-file category.
4. **Description keywords** — weakest signal, use last and treat as a tiebreaker
   between categories the stronger signals already narrowed down to.

## Magic bytes / `file` output → category

| Signature / `file` says | Category |
|---|---|
| `ELF ... executable` | reverse or pwn — run `checksec` immediately to tell which: mitigations present + network target → pwn; standalone analysis task → reverse |
| `PE32`/`PE32+ executable` | reverse or malware-analysis — check description framing (challenge vs. "identify this sample") |
| `PNG image`, `JPEG image`, `GIF image`, `RIFF (WAVE)` | steganography (default) or forensics if the description frames it as "recover a file" rather than "find the hidden message" |
| `Zip archive`, `gzip compressed`, `7-zip archive`, `POSIX tar archive` | forensics (extract and re-classify contents) |
| `PDF document` | forensics (metadata/embedded objects) or steganography |
| `pcap capture file`, `pcapng` | forensics/network |
| `ASCII text` / `data` (no clear magic) with base64/hex-looking content | cryptography |
| Disk/memory image (`DOS/MBR boot sector`, raw `.mem`/`.raw`/`.vmem`) | forensics — memory image → volatility3 |

## Description-keyword → category (tiebreaker only)

| Keywords | Category |
|---|---|
| sql, xss, ssrf, cookie, session, admin panel, login, http | web-exploitation |
| encrypt, cipher, key, modulus, RSA, XOR, hash, decode | cryptography |
| disassemble, decompile, obfuscated, packed, "what does this program do" | reverse-engineering |
| overflow, canary, shellcode, ROP, `nc <host> <port>` + a binary attached | binary-exploitation |
| memory dump, pcap, disk image, metadata, timeline | forensics |
| hidden message, LSB, spectrogram, "something's off with this image" | steganography |
| username, handle, profile, leaked, find the person, social media | osint |
| sample, IOC, malicious, "identify this malware" | malware-analysis |
| `ssh user@host`, `nc host port` with no attached file | live connection — see `tool-usage` skill for ssh/netcat handling |

## When signals disagree

State the conflict explicitly, pick the strongest-signal category, and note the
runner-up. If the first category's tools turn up nothing in 2-3 steps, reconsider
the runner-up rather than continuing to grind — this is exactly the failure mode
the old ladder had no mechanism to recover from.
