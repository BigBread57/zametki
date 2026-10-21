# Windows Event Log Analysis

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/forensics-dfir/windows-event-logs.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/forensics-dfir/windows-event-logs.md)

---

---
name: windows-event-logs
category: forensics-dfir
triggers: ["evtx", "event log", "Security.evtx", "Event ID", "4624", "4625", "4688", "4720", "7045", "1102", "sysmon", "4104", "powershell log", "logon type", "lateral movement", "chainsaw", "hayabusa", "EvtxECmd"]
tools: ["EvtxECmd", "chainsaw", "hayabusa", "Get-WinEvent", "evtx_dump", "python-evtx", "Timeline Explorer", "DeepBlueCLI", "Event Log Explorer"]
priority: high
description: Parse Windows EVTX logs and interpret the key Security, System, and Sysmon Event IDs to reconstruct logons, execution, persistence, and lateral movement.
---

# Windows Event Log Analysis

## Trigger context

Artifact is one or more `.evtx` files (magic `ElfFile\x00`) - `Security.evtx`,
`System.evtx`, `Microsoft-Windows-Sysmon%4Operational.evtx`,
`Microsoft-Windows-PowerShell%4Operational.evtx`, `...TerminalServices...`,
`...WinRM...`. Asks: "when did the attacker log in", "what account was created",
"what service was installed", "was the log cleared", "trace the lateral
movement", "what PowerShell ran". Use scan-everything tools (chainsaw/hayabusa)
first, then pivot to EvtxECmd + Timeline Explorer for detailed reading.

## Objective

Convert raw EVTX into an attacker-activity timeline by decoding the high-value
Event IDs: authentication (who/where/how), process creation (what ran),
account/service changes (persistence), and log tampering - then map each to
MITRE ATT&CK and hand a clean chronology to the Sherlock/incident file.

## Key Event IDs (memorise these)

```
Security.evtx (auditing)
  4624  Successful logon        -> READ THE LOGON TYPE (see table below)
  4625  Failed logon            -> brute force / password spray; check Status codes
  4634 / 4647  Logoff
  4672  Special privileges assigned at logon (admin/SeDebug) -> privileged session
  4648  Logon with explicit credentials (runas / lateral with creds)
  4688  Process creation (needs audit on; incl. command line if configured)
  4689  Process termination
  4720  User account CREATED    | 4722 enabled | 4723/4724 pw change/reset
  4728/4732/4756  User added to (global/local/universal) privileged group
  4726  User account deleted
  4768  Kerberos TGT requested  | 4769 service ticket (kerberoasting)  | 4771 pre-auth fail
  4776  NTLM authentication attempt
  1102  SECURITY LOG CLEARED    <- almost always attacker anti-forensics
  4698  Scheduled task created  | 4699 deleted   (persistence)
  4697  Service installed (Security channel equivalent of System 7045)
System.evtx
  7045  New service installed   <- PsExec/Cobalt Strike/malware persistence
  7034/7036  Service crashed / state change
  104   Log cleared (System channel)
  1074  Shutdown/restart initiated
Sysmon (Microsoft-Windows-Sysmon/Operational) - richest source if present
  1   Process creation (hashes, cmdline, parent) -> best execution evidence
  3   Network connection (proc -> dst IP/port)   -> C2 / lateral
  7   Image/DLL loaded (unsigned/odd DLLs)        -> injection
  8   CreateRemoteThread                          -> injection
  10  ProcessAccess (LSASS access = credential dumping!)
  11  File created                                -> dropped payloads
  12/13/14  Registry create/set/rename            -> persistence
  13  Registry value set (Run keys, etc.)
  15  FileCreateStreamHash (ADS / mark-of-web)
  22  DNS query                                   -> C2 domain resolution
  23/26 File delete                               -> anti-forensics
PowerShell/Operational
  4104  Script block logging (DECODED script content) <- deobfuscated payloads
  4103  Module/pipeline logging
  400/403/600  Engine start/stop, provider (v2 downgrade abuse)
WinRM / TerminalServices / SMB
  4624 type 3 (network) + 5985/5986 -> WinRM lateral
  4624 type 10 (RemoteInteractive) + TerminalServices 1149 -> RDP
```

### Logon Type map (field of 4624/4625)

```
2  Interactive (at keyboard)          9  NewCredentials (runas /netonly)
3  Network (SMB, shares, WMI)        10  RemoteInteractive (RDP)
4  Batch (scheduled task)            11  CachedInteractive (cached domain creds)
5  Service (service account)
7  Unlock (workstation unlock)
Type 3 + admin group + odd source host = lateral movement.
Type 10 = RDP session. Type 9 = often lateral tooling / pass-the-hash-ish.
```

## Commands & payloads

```bash
LOGS=./winevt_logs      # directory of .evtx

# --- Fast triage: scan everything against Sigma + built-in detections --------
chainsaw hunt $LOGS -s sigma/ --mapping mappings/sigma-event-logs-all.yml \
         -r rules/ --level high --csv --output chainsaw_out
chainsaw search "mimikatz" -i $LOGS                 # keyword/regex sweep
chainsaw search -t 'Event.System.EventID: =4624' $LOGS   # Tau field query

hayabusa csv-timeline -d $LOGS -o hayabusa.csv --RFC-3339 -p standard
hayabusa logon-summary -d $LOGS                     # 4624/4625 who/where matrix
hayabusa search -k "powershell" -d $LOGS

# --- Detailed parse (Eric Zimmerman) -> Timeline Explorer --------------------
EvtxECmd.exe -d $LOGS --csv out --csvf events.csv   # normalises all channels
EvtxECmd.exe -f Security.evtx --csv out             # single file

# --- Targeted queries --------------------------------------------------------
# PowerShell (native, on a live/mounted host):
Get-WinEvent -Path Security.evtx -FilterXPath "*[System[EventID=4624]]" |
  Select-Object TimeCreated, @{n='User';e={$_.Properties[5].Value}},
    @{n='LogonType';e={$_.Properties[8].Value}},
    @{n='SrcIP';e={$_.Properties[18].Value}} | Format-Table -Auto
Get-WinEvent -Path Security.evtx -FilterHashtable @{Id=4625} | measure   # failed count
Get-WinEvent -Path 'Microsoft-Windows-PowerShell%4Operational.evtx' `
  -FilterXPath "*[System[EventID=4104]]" | % { $_.Properties[2].Value }  # script blocks

# Linux / cross-platform dump then grep:
evtx_dump Security.evtx > security.xml            # (omerbenamram/evtx, Rust)
python3 -m evtx.scripts.evtx_dump Security.evtx | grep -A5 '4720'

# Anti-forensics quick check:
chainsaw search -t 'Event.System.EventID: =1102' $LOGS   # security log cleared
```

## Branching logic

- **Sysmon present** → prioritise it; Sysmon 1 gives hashes+cmdline+parent that
  Security 4688 often lacks, and Sysmon 10 (LSASS access) directly flags cred
  dumping. Absent → lean on 4688 (if command-line auditing was enabled) + 4104.
- **Brute force question** → cluster 4625 by SourceIP/account; the eventual 4624
  from the same source is the successful compromise; note the Status/SubStatus.
- **Lateral movement** → chain: 4624 type 3/10 on target + source workstation +
  4672 (privs) + 7045/4697 (service drop = PsExec) + Sysmon 3 (net conn).
- **Persistence** → 4720/4732 (rogue admin), 7045/4697 (service), 4698 (task),
  Sysmon 13 Run keys.
- **Log cleared (1102/104)** → note the gap; pivot to other sources (Sysmon,
  forwarded logs, memory, disk artifacts) to cover the blind spot.
- **Obfuscated PowerShell** → 4104 stores the *decoded* script block; concatenate
  multi-part blocks by ScriptBlockId before analysing.

## Common pitfalls

- Timezone: EVTX stores UTC; tools may localise. Fix one timezone for the whole
  timeline and state it.
- Assuming 4688 has command lines - it only does if "Include command line in
  process creation events" auditing was enabled; otherwise use Sysmon 1.
- Reading 4624 without the logon type - a type 5 service logon is not a human.
- Treating every 4625 as an attack - typos exist; look for volume/spray patterns.
- Ignoring the account context: `ANONYMOUS LOGON`, machine accounts (`$`), and
  well-known SIDs change the meaning.
- Missing multi-block 4104 scripts (large payloads split across events).
- Forgetting that a 1102 means data is *gone* - the absence is itself evidence.

## Proof / verification

- Every attacker action = an Event ID + timestamp (UTC) + account + source host.
- Corroborate execution across channels (Sysmon 1 vs Security 4688 vs PowerShell
  4104) before asserting.
- Map each finding to MITRE ATT&CK (e.g. 7045 → T1543.003; 4104 → T1059.001;
  Sysmon 10 on lsass → T1003.001; 1102 → T1070.001).
- Report should reproduce: name the file, Event ID, and record/EventRecordID.

## Cross-references

- [sa--instructions--forensics-dfir--log-analysis-sherlock.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--log-analysis-sherlock.md) - fold these events into the master incident timeline.
- [sa--instructions--forensics-dfir--threat-hunting-detection.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--threat-hunting-detection.md) - Sigma rules that fire on these Event IDs.
- [sa--instructions--forensics-dfir--soc-alert-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--soc-alert-triage.md) - ATT&CK mapping and escalation of a logged detection.
- [sa--instructions--forensics-dfir--memory-volatility.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--memory-volatility.md) / [sa--instructions--forensics-dfir--disk-image-analysis.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--disk-image-analysis.md) - host artifacts to corroborate.

## Credible sources

- Microsoft Security auditing / Event ID reference: https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/basic-audit-logon-events
- 4624 logon type documentation: https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4624
- Sysmon (Sysinternals) & event schema: https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
- Chainsaw: https://github.com/WithSecureLabs/chainsaw
- Hayabusa: https://github.com/Yamato-Security/hayabusa
- EvtxECmd (Eric Zimmerman): https://ericzimmerman.github.io/
- SANS "Hunt Evil" / lateral movement poster: https://www.sans.org/posters/hunt-evil/
- MITRE ATT&CK: https://attack.mitre.org/
