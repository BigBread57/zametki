# Memory Forensics with Volatility 3

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/forensics-dfir/memory-volatility.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/forensics-dfir/memory-volatility.md)

---

---
name: memory-volatility
category: forensics-dfir
triggers: ["memory dump", "volatility", "volatility3", "RAM image", "vmem", "process injection", "malfind", "hashdump", "pslist", "netscan", "hiberfil", "MEMORY.DMP", "crash dump", "credential dump"]
tools: ["volatility3 (vol.py)", "volatility2", "strings", "yara", "bulk_extractor", "MemProcFS", "impacket-secretsdump", "hashcat", "john"]
priority: high
description: Volatility 3 driven RAM analysis - enumerate processes, hunt injected code and rogue network connections, dump processes/files, and recover credentials and flags from a Windows memory image.
---

# Memory Forensics with Volatility 3

## Trigger context

The artifact is a raw physical memory image (`.raw`, `.mem`, `.vmem`, `.dmp`,
`.lime`, `.crash`, `hiberfil.sys`, `MEMORY.DMP`) confirmed by [sa--instructions--forensics-dfir--forensics-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--forensics-triage.md).
Typical asks: "what process was malicious?", "find the injected shellcode",
"recover the password / flag from RAM", "what did the attacker run?", "what was
the C2 IP?". Volatility 3 is the default engine; drop to Volatility 2 only for
legacy/Linux profiles it still handles better.

## Objective

Reconstruct the live-system state frozen in the image: running/hidden processes,
their command lines and parents, network connections, injected regions, loaded
drivers, and on-disk/in-RAM secrets - and extract the specific artifact the
challenge wants (flag string, credential, dropped binary, C2 address).

## Methodology (decision tree)

```
1. IDENTIFY   windows.info  (build, arch, KdVersionBlock) - Vol3 auto-detects,
              no manual profile. If it errors: wrong OS family or truncated image.
2. PROCESSES  windows.pslist -> windows.pstree (parentage) -> windows.psscan
              (unlinked/hidden) -> diff pslist vs psscan for rootkit hiding.
3. WHAT RAN   windows.cmdline, windows.consoles, windows.cmdscan, windows.envars
4. NETWORK    windows.netscan (and netstat) -> foreign IPs, listeners, C2.
5. INJECTION  windows.malware.malfind -> MZ / PAGE_EXECUTE_READWRITE / 0x90 sleds.
              windows.malware.ldrmodules (unlinked DLLs),
              windows.malware.hollowprocesses / processghosting.
6. PERSIST    windows.registry.printkey (Run keys), windows.svcscan,
              windows.registry.scheduled_tasks.
7. EXTRACT    windows.dumpfiles, windows.pslist --dump, windows.memmap --dump,
              windows.vadinfo/vaddump for a single region.
8. SECRETS    windows.hashdump, windows.registry.lsadump, windows.cachedump,
              then crack offline OR pass-the-hash.
9. CORRELATE  build a timeline; tie the rogue PID to its cmdline, its netconn,
              its injected VAD, and its persistence.
```

## Commands & payloads

```bash
# Vol3 install: pipx install volatility3   (invoke as `vol` or `python3 vol.py`)
IMG=memdump.raw

# --- 1. Identify -------------------------------------------------------------
vol -f $IMG windows.info                 # OS build/arch; confirms it's parseable
vol -f $IMG banners.Banners              # Linux/Mac: recover kernel banner for ISF

# --- 2. Processes ------------------------------------------------------------
vol -f $IMG windows.pslist
vol -f $IMG windows.pstree               # READ THIS FIRST - parentage reveals evil
vol -f $IMG windows.psscan               # pool scan; catches terminated/hidden
# Rogue signs: cmd/powershell child of a Word/Outlook/browser PID;
# svchost NOT parented by services.exe; lsass with a non-wininit parent;
# misspelled names (scvhost, lsass.exe from wrong path), PID reuse.

# --- 3. Execution evidence ---------------------------------------------------
vol -f $IMG windows.cmdline              # full command line per PID
vol -f $IMG windows.consoles            # console input/output buffers (attacker typing)
vol -f $IMG windows.cmdscan             # scanned command history
vol -f $IMG windows.envars --pid 1234

# --- 4. Network --------------------------------------------------------------
vol -f $IMG windows.netscan             # TCP/UDP endpoints, owning PID, state
vol -f $IMG windows.netstat
# Pivot: any ESTABLISHED to a public IP, or LISTENING on an odd high port,
# owned by an injected/suspicious PID = C2.

# --- 5. Injection / malware --------------------------------------------------
vol -f $IMG windows.malware.malfind     # (older builds: windows.malfind)
vol -f $IMG windows.malware.malfind --dump   # dump each suspicious VAD to file
vol -f $IMG windows.malware.ldrmodules  # DLLs missing from one of the 3 lists = injected
vol -f $IMG windows.malware.hollowprocesses
vol -f $IMG windows.dlllist --pid 1234
vol -f $IMG windows.vadinfo --pid 1234  # then vaddump a specific PAGE_EXECUTE_RW range
vol -f $IMG windows.malware.malfind --pid 1234 | grep -i 'MZ\|PAGE_EXEC'

# --- 6. Persistence ----------------------------------------------------------
vol -f $IMG windows.registry.hivelist
vol -f $IMG windows.registry.printkey --key "Software\Microsoft\Windows\CurrentVersion\Run"
vol -f $IMG windows.svcscan
vol -f $IMG windows.registry.scheduled_tasks

# --- 7. Extraction -----------------------------------------------------------
vol -f $IMG windows.dumpfiles --pid 1234                 # all mapped files for a PID
vol -f $IMG windows.dumpfiles --virtaddr 0xVADBASE       # one region
vol -f $IMG -o ./out windows.pslist --pid 1234 --dump    # dump the process image
vol -f $IMG -o ./out windows.memmap --pid 1234 --dump    # full process address space
vol -f $IMG windows.filescan | grep -i 'flag\|\.docx\|\.zip\|desktop'
vol -f $IMG windows.dumpfiles --physaddr 0xFILEOFFSET

# --- 8. Credentials ----------------------------------------------------------
vol -f $IMG windows.hashdump             # SAM NTLM hashes (user:rid:LM:NT)
vol -f $IMG windows.registry.lsadump      # LSA secrets (service creds, cached)
vol -f $IMG windows.registry.cachedump    # domain cached creds (mscash)
# Then offline:
hashcat -m 1000 nt_hashes.txt rockyou.txt         # NTLM
hashcat -m 2100 mscash2.txt   rockyou.txt         # DCC2 / domain cached
# lsass captured as a dump instead? -> impacket/pypykatz on the dumped lsass.exe

# --- 9. Flag / string sweep --------------------------------------------------
strings -e l $IMG | grep -Ei 'flag\{|htb\{|ctf\{'      # UTF-16LE first on Windows
strings     $IMG | grep -Ei 'flag\{|password='
vol -f $IMG windows.strings --strings-file wanted.txt  # map a string back to PID/offset
```

## Branching logic

- **`windows.info` errors / empty** → image truncated, wrong OS, or it is a
  hibernation/crash file needing conversion. For hiberfil.sys use a recent Vol3
  build (native support) or convert; for `MEMORY.DMP` Vol3 reads it directly.
- **Linux/macOS image** → you need a matching ISF symbol table. Recover the
  kernel banner with `banners.Banners`, build/download the ISF, then use
  `linux.*` / `mac.*` plugins (pslist, bash, check_creds, malfind).
- **pslist and psscan disagree** → a process is unlinked (DKOM hiding); trust
  psscan and investigate that PID hard.
- **malfind hits but process looks legit** → likely hollowing/injection into a
  trusted process; confirm with hollowprocesses + ldrmodules, then dump the VAD
  and analyse the shellcode/PE offline.
- **Need the dropped payload** → filescan to find it, dumpfiles by physaddr,
  then run [sa--instructions--forensics-dfir--forensics-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--forensics-triage.md) Step 0 on the extracted file.
- **Creds recovered** → decide crack (hashcat) vs reuse (pass-the-hash) based on
  the challenge objective; a flag is usually the cracked plaintext.

## Common pitfalls

- Using a Vol2 profile mindset in Vol3 - there is no `--profile`; Vol3 auto-IDs.
- Plugin path drift: modern Vol3 nests malware plugins under `windows.malware.*`
  (e.g. `windows.malware.malfind`). If `windows.malfind` errors, try the
  `malware.` prefix and run `vol -f $IMG -h` to list what your build exposes.
- Reporting a PID as evil without its command line, parent, and a second
  corroborating signal (netconn or injected VAD).
- Forgetting UTF-16LE: Windows flags/passwords in RAM are usually wide chars.
- Dumping into the working dir without `-o`; outputs collide.
- Assuming netscan connections are live - they are residue; correlate with the
  owning PID's state and the process's other evidence.
- Cracking hashes you did not need when the objective was lateral movement.

## Proof / verification

- Chain the rogue PID end to end: `pstree` parentage + `cmdline` + `netscan`
  foreign IP + `malfind` injected region + persistence key.
- Hash every dumped process/file; verify a dumped PE has a valid MZ/PE header.
- Cross-check hashdump output against known accounts; a cracked password that
  logs in (or matches the flag format) is the verification.
- State the Vol3 version and exact plugin names used so the run reproduces.

## Cross-references

- [sa--instructions--forensics-dfir--forensics-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--forensics-triage.md) - confirming the artifact is a memory image and routing.
- [sa--instructions--forensics-dfir--disk-image-analysis.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--disk-image-analysis.md) - for hiberfil.sys / pagefile.sys pulled from a disk.
- [sa--instructions--forensics-dfir--windows-event-logs.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--windows-event-logs.md) - corroborate in-RAM activity with logged Event IDs.
- [sa--instructions--forensics-dfir--log-analysis-sherlock.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--log-analysis-sherlock.md) - fold memory findings into the incident timeline.

## Credible sources

- Volatility 3 documentation & plugin index: https://volatility3.readthedocs.io/en/latest/
- Volatility 3 Windows plugins list: https://volatility3.readthedocs.io/en/latest/volatility3.plugins.windows.html
- Volatility 3 GitHub: https://github.com/volatilityfoundation/volatility3
- Volatility CheatSheet (basics): https://blog.onfvp.com/post/volatility-cheatsheet/
- HackTricks memory dump analysis: https://book.hacktricks.wiki/en/forensics/basic-forensic-methodology/memory-dump-analysis/index.html
- SANS memory forensics poster: https://www.sans.org/posters/memory-forensics-cheat-sheet/
- hashcat example hashes (modes): https://hashcat.net/wiki/doku.php?id=example_hashes
