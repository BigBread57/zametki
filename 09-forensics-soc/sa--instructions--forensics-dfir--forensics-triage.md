# Forensics Triage & Artifact Routing

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/forensics-dfir/forensics-triage.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/forensics-dfir/forensics-triage.md)

---

---
name: forensics-triage
category: forensics-dfir
triggers: ["forensics", "unknown file", "artifact identification", "file carving", "binwalk", "foremost", "exiftool", "steganography", "what kind of file is this", "triage", "hex dump", "magic bytes"]
tools: ["file", "binwalk", "foremost", "scalpel", "photorec", "exiftool", "strings", "xxd", "hexdump", "7z", "trid", "bulk_extractor", "yara", "ssdeep", "python-magic"]
priority: high
description: First-contact routing layer for any forensics artifact - identify the container type, extract embedded data, and dispatch to the correct specialist methodology (memory / disk / pcap / logs).
---

# Forensics Triage & Artifact Routing

## Trigger context

Use this file **first** whenever a forensics or DFIR challenge hands you an opaque
artifact: `evidence.bin`, `dump.raw`, `challenge.zip`, `image.jpg`, `capture`,
`suspicious.doc`, an unnamed blob, or a HackTheBox Sherlock archive. It is the
dispatcher: it answers "what am I holding?" and then hands off to the correct
deep-dive methodology file. Do not start Volatility or Wireshark until this file
has confirmed the artifact class.

## Objective

1. Positively identify the artifact type from content, never from the extension.
2. Recover everything nested inside it (appended archives, carved files, metadata).
3. Route to the correct specialist file with a one-line justification.
4. Produce a hash-anchored evidence inventory so later findings are reproducible.

## Methodology (decision tree)

```
artifact
├── Step 0: hash + size + `file` + first 64 bytes hexdump   (ALWAYS)
│
├── magic = archive (PK.., Rar!, 7z¼¯', ustar, \x1f\x8b)
│      └── extract recursively; watch for nested/password-protected layers
│          → password? try: filename, "infected", README, challenge prompt
│
├── magic = image/audio/video (JPEG FFD8FF, PNG 89504E47, GIF, RIFF, ID3)
│      └── exiftool → strings → binwalk → stego path
│          (steghide/zsteg/stegsolve/LSB) - see crypto/stego notes
│
├── magic = ELF / PE / Mach-O
│      └── this is reverse engineering, not forensics → route to RE methodology
│
├── magic = PDF / OLE2 (D0CF11E0) / OOXML (PK with word/, xl/)
│      └── maldoc path: oledump.py, olevba, pdfid/pdf-parser, extract macros/IOCs
│
├── magic = pcap (D4C3B2A1 / A1B2C3D4) or pcapng (0A0D0D0A)
│      └── → pcap-network-forensics.md
│
├── magic = EVTX ("ElfFile\x00") or .evtx/.evt extension
│      └── → windows-event-logs.md
│
├── size is a round multiple of RAM (512MB/1GB/2GB/4GB/8GB), `file` says "data",
│   strings show "PAGEDU64"/"KDBG"/"Pagefile"/kernel paths, or extension
│   .raw/.mem/.vmem/.dmp/.lime/.crash
│      └── → memory-volatility.md
│
├── magic = EWF ("EVF\x09\x0d\x0a\xff\x00") / .E01, or raw with MBR 55AA @0x1FE,
│   or GPT "EFI PART" @0x200, or NTFS/FAT/EXT superblock
│      └── → disk-image-analysis.md
│
├── plain text: JSON/CSV/syslog/IIS/Apache/Zeek/Sysmon-JSON/Sherlock bundles
│      └── → log-analysis-sherlock.md (and soc-alert-triage.md if alert context)
│
└── none of the above / entropy ~7.99
       └── encrypted or compressed blob: carve, brute the wrapper, check for
           XOR/base64 layering, then re-run Step 0 on the recovered output
```

**Recursion rule:** every time you extract something, restart at Step 0 on the
output. CTF forensics artifacts are usually 2-4 layers deep.

## Commands & payloads

```bash
# --- Step 0: always run this block first -------------------------------------
sha256sum evidence.bin && ls -l evidence.bin
file evidence.bin
xxd evidence.bin | head -n 8          # confirm magic bytes by eye
trid evidence.bin                     # probabilistic type ID when `file` fails
python3 -c "import math,collections,sys;d=open(sys.argv[1],'rb').read();\
c=collections.Counter(d);print(round(-sum(v/len(d)*math.log2(v/len(d)) for v in c.values()),3))" evidence.bin
# entropy > 7.9 => encrypted/compressed; ~5-6 => text/executable

# --- Strings triage (fast flag / IOC sweep) ----------------------------------
strings -n 8 evidence.bin | head -n 200
strings -e l evidence.bin | grep -Ei 'flag|htb|ctf\{|password|BEGIN .*PRIVATE'
strings -a evidence.bin | grep -Eo '([0-9]{1,3}\.){3}[0-9]{1,3}' | sort -u
strings -a evidence.bin | grep -Eio 'https?://[a-z0-9./?=_%:-]{6,}' | sort -u

# --- Embedded / appended data ------------------------------------------------
binwalk evidence.bin                     # signature scan
binwalk -e --dd='.*' evidence.bin        # extract everything it recognises
binwalk -Me evidence.bin                 # recursive matryoshka extraction
7z l evidence.bin && 7z x evidence.bin -o_out   # 7z reads more formats than unzip

# --- Carving unallocated / raw blobs -----------------------------------------
foremost -t all -i evidence.bin -o carved_out
scalpel -c /etc/scalpel/scalpel.conf -o scalpel_out evidence.bin
photorec evidence.bin                    # interactive, best recall on disk images
bulk_extractor -o bulk_out evidence.bin  # emails, URLs, CCNs, PII, AES keys

# --- Metadata ----------------------------------------------------------------
exiftool -a -u -g1 photo.jpg             # all tags, unknown tags, grouped
exiftool -ee -a suspicious.pdf           # extract embedded/duplicate blocks
exiftool -Comment -UserComment -GPS* *.jpg
mediainfo video.mp4

# --- Office / PDF maldocs -----------------------------------------------------
oledump.py suspicious.doc                # list streams; A/M marks macro streams
oledump.py -s 8 -v suspicious.doc        # decompress VBA stream 8
olevba --decode suspicious.docm
pdfid.py doc.pdf                         # /JS /OpenAction /Launch /EmbeddedFile
pdf-parser.py --object 12 --filter doc.pdf

# --- Archive password recovery (last resort, wordlist first) ------------------
zip2john locked.zip > hash.txt && john --wordlist=rockyou.txt hash.txt
7z2john locked.7z  > hash.txt && john --wordlist=rockyou.txt hash.txt

# --- Similarity / known-bad ---------------------------------------------------
ssdeep -b evidence.bin
yara -r rules/malware_index.yar evidence.bin
```

## Branching logic

| Observation | Branch to |
|---|---|
| `file` = "pcap capture file" / pcapng | [sa--instructions--forensics-dfir--pcap-network-forensics.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--pcap-network-forensics.md) |
| Strings contain `\SystemRoot\System32`, `KDBG`, `PsActiveProcessHead`, size ≈ RAM | [sa--instructions--forensics-dfir--memory-volatility.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--memory-volatility.md) |
| `EVF`/`E01`, MBR `55 AA`, `EFI PART`, NTFS `$MFT` | [sa--instructions--forensics-dfir--disk-image-analysis.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--disk-image-analysis.md) |
| `ElfFile` magic / `.evtx` | [sa--instructions--forensics-dfir--windows-event-logs.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--windows-event-logs.md) |
| Directory of mixed logs (Sysmon JSON, IIS, EDR CSV) | [sa--instructions--forensics-dfir--log-analysis-sherlock.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--log-analysis-sherlock.md) |
| Single alert + IOC list to judge | [sa--instructions--forensics-dfir--soc-alert-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--soc-alert-triage.md) |
| Broad "find evil in this dataset" with no alert | [sa--instructions--forensics-dfir--threat-hunting-detection.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--threat-hunting-detection.md) |
| PE/ELF with packed sections | reverse-engineering methodology |
| Image/audio with clean metadata but odd size | steganography methodology |

Ambiguity rule: if two branches are plausible (e.g. a hibernation file inside a
disk image), take the **disk** branch first to extract, then run the **memory**
branch on the extracted `hiberfil.sys` / `pagefile.sys` / `MEMORY.DMP`.

## Common pitfalls

- Trusting the extension. Challenge authors rename `.jpg` onto a pcap constantly.
- Running `unzip` and stopping: `unzip` ignores data appended after the central
  directory; `binwalk -Me` and `7z` do not.
- Forgetting `strings -e l` (UTF-16LE). Windows artifacts store most useful
  strings wide, and plain `strings` misses them entirely.
- Carving into the same directory as the source, then re-carving your own output
  in an infinite loop. Always `-o` to a fresh directory.
- Ignoring slack/trailing bytes: check `ls -l` against the parsed structure size.
- Overwriting evidence in place. Work on a copy; keep the original hash.
- Treating high entropy as "nothing here" - it usually means one more layer.
- Skipping `exiftool` on non-images: it parses PDFs, Office docs, and videos too.

## Proof / verification

- Record `sha256` of the original artifact and of every extracted child.
- Every claim about type must cite the magic bytes you saw, not `file` alone.
- After carving, re-run `file` on each carved object; discard zero-byte and
  obviously false-positive carves before reporting.
- The final answer must name the routing decision explicitly:
  "artifact = Windows raw memory image (KDBG present at offset X) → Volatility 3".

## Cross-references

- [sa--instructions--forensics-dfir--memory-volatility.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--memory-volatility.md) - RAM images, injected code, credential recovery.
- [sa--instructions--forensics-dfir--disk-image-analysis.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--disk-image-analysis.md) - E01/dd mounting, `$MFT`, deleted-file recovery.
- [sa--instructions--forensics-dfir--pcap-network-forensics.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--pcap-network-forensics.md) - packet captures, file/credential extraction.
- [sa--instructions--forensics-dfir--windows-event-logs.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--windows-event-logs.md) - EVTX parsing and Event ID semantics.
- [sa--instructions--forensics-dfir--log-analysis-sherlock.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--log-analysis-sherlock.md) - multi-source timeline building.
- [sa--instructions--forensics-dfir--soc-alert-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--soc-alert-triage.md) - when the artifact arrives attached to an alert.

## Credible sources

- Volatility 3 documentation: https://volatility3.readthedocs.io/en/latest/
- binwalk: https://github.com/ReFirmLabs/binwalk
- ExifTool: https://exiftool.org/
- foremost man page: https://linux.die.net/man/1/foremost
- bulk_extractor: https://github.com/simsong/bulk_extractor
- Didier Stevens oledump/pdf-parser: https://blog.didierstevens.com/programs/oledump-py/
- HackTricks forensics methodology: https://book.hacktricks.wiki/en/forensics/basic-forensic-methodology/index.html
- SANS DFIR cheat sheets: https://www.sans.org/posters/?focus-area=digital-forensics
- File signature (magic byte) table: https://en.wikipedia.org/wiki/List_of_file_signatures
