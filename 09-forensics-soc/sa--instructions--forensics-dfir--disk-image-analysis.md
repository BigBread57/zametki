# Disk Image Analysis

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/forensics-dfir/disk-image-analysis.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/forensics-dfir/disk-image-analysis.md)

---

---
name: disk-image-analysis
category: forensics-dfir
triggers: ["disk image", "E01", "dd image", "raw image", "autopsy", "sleuthkit", "MFT", "$MFT", "deleted file recovery", "registry hive", "browser history", "filesystem timeline", "NTFS", "shellbags", "prefetch", "amcache"]
tools: ["ewfmount", "mount", "affuse", "The Sleuth Kit (mmls/fls/icat/tsk_recover)", "autopsy", "MFTECmd", "RECmd", "EvtxECmd", "PECmd", "AmcacheParser", "ShellBags Explorer", "log2timeline/plaso", "regripper", "bulk_extractor"]
priority: high
description: Mount and dissect Windows/Linux disk images - build a filesystem timeline, recover deleted files, and mine registry, browser, and execution artifacts for the investigative answer.
---

# Disk Image Analysis

## Trigger context

Confirmed by [sa--instructions--forensics-dfir--forensics-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--forensics-triage.md): an `.E01`/`.Ex01` (EWF), a `.dd`/`.raw`/`.img`
(raw), a `.vmdk`/`.vhdx`, or a mounted volume. Asks look like: "recover the
deleted file", "what was downloaded", "when was the malware installed",
"reconstruct user activity", "find the flag hidden in unallocated space". This is
the dead-box branch; pair with [sa--instructions--forensics-dfir--windows-event-logs.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--windows-event-logs.md) for the logs living on it.

## Objective

Turn a static block device into an answerable timeline: what files existed
(including deleted), what executed, what the user did, and what the attacker
touched - anchored to timestamps and to specific artifacts (`$MFT`, registry,
browser DBs, prefetch/amcache).

## Methodology (decision tree)

```
1. IMAGE PREP   verify hash; E01 -> ewfmount to a raw view; note segments.
2. PARTITIONS   mmls (partition table) -> pick offset -> fsstat (fs type/details).
3. MOUNT RO     loop-mount read-only,noexec at the sector offset, OR work
                offset-based with TSK/Autopsy (no mount needed).
4. TRIAGE       KAPE/Autopsy ingest, OR targeted artifact pull:
                  $MFT, $LogFile, $UsnJrnl, registry hives, Event logs,
                  prefetch, amcache, browser profiles, $Recycle.Bin.
5. TIMELINE     MFTECmd ($MFT) + plaso super-timeline -> single chronology.
6. DELETED      fls -rd (deleted names) -> icat by inode, OR tsk_recover,
                OR photorec/foremost carving on unallocated.
7. REGISTRY     RECmd/regripper: Run keys, UserAssist, ShellBags, USB, timezone,
                RecentDocs, typed paths, services, network.
8. EXECUTION    PECmd (prefetch), AmcacheParser (amcache.hve), SrumECmd (SRUM),
                AppCompatCache/Shimcache.
9. USER/WEB     browser history/downloads/cookies (SQLite), LNK/JumpLists,
                Recent, mailboxes.
10. ANSWER      pivot every finding onto the timeline; tie file->execution->user.
```

## Commands & payloads

```bash
# --- 1-3. Mount --------------------------------------------------------------
sha256sum disk.E01
ewfmount disk.E01 /mnt/ewf            # exposes /mnt/ewf/ewf1 as a raw device
mmls /mnt/ewf/ewf1                    # partition table; note the Start sector
fsstat -o 2048 /mnt/ewf/ewf1         # filesystem details at that offset
# byte offset = start_sector * 512 (mmls sectors are 512B here => 2048*512)
mount -o ro,loop,noexec,offset=$((2048*512)) /mnt/ewf/ewf1 /mnt/disk
# raw dd instead of E01:
affuse disk.dd /mnt/aff   # or just loop-mount the .dd directly

# --- 5/6. Sleuth Kit timeline + deleted files --------------------------------
fls -r -o 2048 /mnt/ewf/ewf1                  # recursive listing; * marks deleted
fls -rd -o 2048 /mnt/ewf/ewf1                 # deleted entries only
icat -o 2048 /mnt/ewf/ewf1 <inode> > recovered.bin
tsk_recover -a -o 2048 /mnt/ewf/ewf1 ./recovered/   # bulk recover allocated+unalloc
# Super timeline (bodyfile -> mactime):
fls -r -m C: -o 2048 /mnt/ewf/ewf1 > body.txt
mactime -b body.txt -d > timeline.csv

# --- Eric Zimmerman tools (Windows; run on the mounted files) ----------------
MFTECmd.exe -f "E:\$MFT" --csv out --csvf mft.csv          # full $MFT
MFTECmd.exe -f "E:\$Extend\$UsnJrnl:$J" --csv out          # USN journal (renames/deletes)
PECmd.exe -d "E:\Windows\Prefetch" --csv out               # what executed + run counts
AmcacheParser.exe -f "E:\Windows\appcompat\Programs\Amcache.hve" --csv out
AppCompatCacheParser.exe -f "E:\...\SYSTEM" --csv out       # Shimcache
SrumECmd.exe -f "E:\Windows\System32\sru\SRUDB.dat" --csv out
EvtxECmd.exe -d "E:\Windows\System32\winevt\Logs" --csv out # -> see windows-event-logs.md
RECmd.exe --bn BatchExamples\Kroll_Batch.reb -d "E:\Windows\System32\config" --csv out
SBECmd.exe -d "E:\Users\bob\..." --csv out                 # ShellBags (folder access)

# --- 7. Registry (Linux, regripper) ------------------------------------------
rip.pl -r /mnt/disk/Windows/System32/config/SOFTWARE -p run
rip.pl -r /mnt/disk/Windows/System32/config/SYSTEM   -p usbstor
rip.pl -r /mnt/disk/Users/bob/NTUSER.DAT -p userassist
# Key registry answers:
#  SYSTEM\CurrentControlSet\Control\ComputerName        -> hostname
#  SYSTEM ...\Control\TimeZoneInformation               -> local time offset
#  SYSTEM ...\Enum\USBSTOR                               -> USB devices
#  SOFTWARE ...\CurrentVersion\Run / RunOnce             -> persistence
#  NTUSER.DAT\...\Explorer\UserAssist                    -> GUI program execution
#  NTUSER.DAT\...\Explorer\RecentDocs / TypedPaths       -> user activity

# --- 9. Browser & user artifacts ---------------------------------------------
sqlite3 /mnt/disk/Users/bob/AppData/.../History \
  "SELECT datetime(last_visit_time/1000000-11644473600,'unixepoch'),url FROM urls ORDER BY 1 DESC LIMIT 50;"
sqlite3 .../History "SELECT target_path,tab_url FROM downloads;"
# Firefox: places.sqlite (moz_places), Chrome/Edge: History (urls, downloads)

# --- Full auto super-timeline (plaso) ----------------------------------------
log2timeline.py --storage-file plaso.db disk.E01
psort.py -o l2tcsv -w super_timeline.csv plaso.db
# then load super_timeline.csv into Timeline Explorer and filter by column
```

## Branching logic

- **E01 vs raw**: E01 needs `ewfmount`/Arsenal Image Mounter first; raw `.dd`
  mounts directly. Always mount `ro,noexec`.
- **Encrypted volume (BitLocker)**: look for the recovery key in another
  artifact/memory image; mount with `dislocker` before proceeding.
- **Question = a deleted file** → `fls -rd` → `icat` by inode; if the MFT record
  is gone, carve unallocated with photorec/foremost.
- **Question = "what ran"** → prefetch (PECmd) + amcache + shimcache + SRUM;
  cross-check the three, they cover different execution evidence.
- **Question = "what the user did"** → UserAssist + ShellBags + RecentDocs +
  LNK/JumpLists + browser DBs.
- **Question = "when installed / infected"** → `$MFT` `$SI` vs `$FN` timestamps
  (mismatch = timestomping) + USN journal + amcache first-seen.
- **Linux image** → mount ext4, then bash_history, /var/log, cron, systemd
  units, authorized_keys, /etc/passwd; route logs to [sa--instructions--forensics-dfir--log-analysis-sherlock.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--log-analysis-sherlock.md).

## Common pitfalls

- Mounting read-write and mutating evidence - always `ro,noexec`, work on a copy.
- Miscomputing the byte offset (mmls gives *sectors*; multiply by sector size).
- Trusting `$STANDARD_INFORMATION` times alone - attackers timestomp them;
  `$FILE_NAME` times are harder to forge, so compare both in MFTECmd output.
- Ignoring the USN journal and `$LogFile` - they retain deletes/renames the MFT
  no longer shows.
- Chrome/WebKit timestamps are microseconds since 1601, not Unix; convert
  (`/1000000 - 11644473600`). FILETIME everywhere on Windows.
- Reading a live/locked hive - copy hives out first (or use VSS/`ese`-aware tools).
- Skipping Volume Shadow Copies: older versions of a file live in VSS snapshots.

## Proof / verification

- Anchor every claim to a specific artifact + timestamp (e.g. "amcache first-seen
  2024-03-11T02:14Z; prefetch run count 3; MFT `$SI` created same minute").
- Recovered files: verify magic/hash and that they open; re-triage them.
- Cross-validate execution across prefetch + amcache + shimcache before asserting.
- Keep the source image hash; note the mount offset so the analysis reproduces.

## Cross-references

- [sa--instructions--forensics-dfir--forensics-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--forensics-triage.md) - image identification and initial carving.
- [sa--instructions--forensics-dfir--windows-event-logs.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--windows-event-logs.md) - EVTX pulled from `Windows\System32\winevt\Logs`.
- [sa--instructions--forensics-dfir--memory-volatility.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--memory-volatility.md) - analysing `hiberfil.sys`/`pagefile.sys`/`MEMORY.DMP`.
- [sa--instructions--forensics-dfir--log-analysis-sherlock.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--log-analysis-sherlock.md) - merging disk artifacts into one incident timeline.

## Credible sources

- The Sleuth Kit docs: https://sleuthkit.org/sleuthkit/docs.php
- Autopsy: https://www.autopsy.com/
- Eric Zimmerman's tools: https://ericzimmerman.github.io/
- MFTECmd guide (SANS): https://www.sans.org/tools/mftecmd/
- Plaso / log2timeline: https://plaso.readthedocs.io/en/latest/
- RegRipper: https://github.com/keydet89/RegRipper3.0
- libewf / ewfmount: https://github.com/libyal/libewf
- HackTricks partitions/file-systems: https://book.hacktricks.wiki/en/generic-methodologies-and-resources/basic-forensic-methodology/partitions-file-systems-carving/index.html
- SANS Windows forensic analysis poster: https://www.sans.org/posters/windows-forensic-analysis/
