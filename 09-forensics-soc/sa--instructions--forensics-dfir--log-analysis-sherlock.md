# Log Analysis & DFIR Investigation (Sherlock methodology)

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/forensics-dfir/log-analysis-sherlock.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/forensics-dfir/log-analysis-sherlock.md)

---

---
name: log-analysis-sherlock
category: forensics-dfir
triggers: ["sherlock", "HackTheBox sherlock", "DFIR investigation", "incident timeline", "who what when how", "pivot on IOC", "correlate logs", "artifact bundle", "investigation questions", "blue team investigation"]
tools: ["Timeline Explorer", "chainsaw", "hayabusa", "EvtxECmd", "MFTECmd", "jq", "grep/ripgrep", "csvkit", "plaso/psort", "Splunk", "Elastic/Kibana", "python/pandas"]
priority: high
description: End-to-end DFIR investigation workflow for HackTheBox Sherlocks and incident bundles - build one timeline across mixed artifacts, answer the who/what/when/where/how, and pivot on IOCs to reconstruct the full attack chain.
---

# Log Analysis & DFIR Investigation (Sherlock methodology)

## Trigger context

You are handed an investigation bundle, not a single artifact: a HackTheBox
"Sherlock" zip, or an incident package mixing EVTX, Sysmon, MFT/registry exports,
PowerShell logs, IIS/Apache/nginx access logs, EDR CSVs, pcap, and firewall/proxy
logs - plus a list of scenario questions ("What was the attacker's source IP?",
"What was the first command executed?", "What time was persistence established?").
This file is the orchestration layer that drives the specialist files and turns
scattered evidence into one answer set.

## Objective

Reconstruct the incident as a single UTC-ordered timeline and answer, with
evidence, the investigative questions:
- **Who** (account, source host/IP, actor)
- **What** (tools, commands, files, malware)
- **When** (initial access → actions on objective, per event)
- **Where** (which hosts, which pivots)
- **How** (technique per step, mapped to MITRE ATT&CK)

## Methodology (decision tree)

```
0. READ THE PROMPT   Let the Sherlock questions define scope; note the flag
                     format each answer wants (IP, ISO8601 UTC, filename, hash).
1. INVENTORY         List every artifact; route each via forensics-triage.md.
                     Record source host + timezone for each log.
2. NORMALISE         Parse each source to CSV/JSONL with a common UTC timestamp
                     column (EvtxECmd, MFTECmd, chainsaw, hayabusa, jq).
3. ANCHOR            Find a fixed point (the alert time, first 4625 burst, the
                     malware drop). Everything is measured relative to it.
4. TIMELINE          Merge normalised rows into ONE super-timeline; sort by UTC.
5. PIVOT ON IOCs     Take each IOC (IP, user, host, hash, filename, PID, domain)
                     and sweep ALL sources for it; every hit is a new lead + IOC.
6. RECONSTRUCT CHAIN Order findings along the kill chain:
                     Initial Access -> Execution -> Persistence -> PrivEsc ->
                     Defense Evasion -> Cred Access -> Discovery -> Lateral ->
                     Collection -> C2 -> Exfil -> Impact.
7. ANSWER            Map each Sherlock question to its evidence row(s); format
                     exactly as asked.
8. VERIFY            Corroborate each key claim with >=2 independent sources.
```

## Commands & payloads

```bash
BUNDLE=./sherlock_artifacts

# --- 1/2. Normalise each source to a common timeline -------------------------
# Windows event logs -> CSV
EvtxECmd.exe -d "$BUNDLE/evtx" --csv out --csvf evtx.csv
hayabusa csv-timeline -d "$BUNDLE/evtx" -o hayabusa.csv --RFC-3339 -p standard
chainsaw hunt "$BUNDLE/evtx" -s sigma/ --mapping mappings/sigma-event-logs-all.yml \
         --csv --output chainsaw_out
# Filesystem / execution
MFTECmd.exe -f "$BUNDLE/\$MFT" --csv out --csvf mft.csv
# Full super-timeline across everything plaso understands
log2timeline.py --storage-file plaso.db "$BUNDLE"
psort.py -o l2tcsv -w super_timeline.csv plaso.db

# --- Web/proxy/firewall access logs -----------------------------------------
# Top talkers, suspicious UA/URIs (adapt fields to the log format)
rg -N 'POST|cmd|whoami|\.php\?|union|\.\./' "$BUNDLE/web" | head
awk '{print $1}' "$BUNDLE/web/access.log" | sort | uniq -c | sort -rn | head   # top src IPs
# JSON logs (EDR/Sysmon-JSON/cloud):
jq -r 'select(.EventID==1) | [.UtcTime,.User,.Image,.CommandLine] | @tsv' events.json

# --- 5. IOC pivot sweep (run each confirmed IOC across ALL text sources) -----
IOC="10.10.14.7"
rg -n --no-heading "$IOC" "$BUNDLE" -g '!*.evtx'          # text logs
chainsaw search "$IOC" -i "$BUNDLE/evtx"                  # binary evtx
grep -rIi "attacker_user\|evil.exe\|<sha256>\|c2.example" "$BUNDLE"

# --- 4/7. Build & query the merged timeline ----------------------------------
# Concatenate normalised CSVs (align a Timestamp column) then load in
# Timeline Explorer; or query with pandas:
python3 - <<'PY'
import pandas as pd, glob
frames=[pd.read_csv(f, dtype=str) for f in glob.glob("out/*.csv")]
df=pd.concat(frames, ignore_index=True)
df['ts']=pd.to_datetime(df.filter(regex='(?i)time').iloc[:,0], utc=True, errors='coerce')
df=df.sort_values('ts')
print(df[(df.ts>='2024-03-11T02:00Z') & (df.ts<='2024-03-11T03:00Z')].to_string())
PY
```

## Branching logic

- **Question wants a source IP / initial access** → web logs (RCE/upload/SQLi
  patterns) + 4625 bursts + firewall; the first success after failures is it.
- **Question wants first command / execution** → Sysmon 1 / 4688 / PowerShell
  4104, ordered by UTC; watch for the parent that spawned the shell.
- **Question wants persistence time** → 7045/4697 (service), 4698 (task), Sysmon
  13 (Run key), new 4720 account - take the earliest.
- **Question wants lateral movement** → 4624 type 3/10 to a new host + service
  drop on target + Sysmon 3 network conn (see [sa--instructions--forensics-dfir--windows-event-logs.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--windows-event-logs.md)).
- **Question wants exfil** → pcap volume/DNS tunnel + proxy bytes-out + archive
  creation (Sysmon 11 of a .zip/.rar) before the transfer.
- **A log is cleared (1102)** → mark the blind window; cover it from another
  source (memory, MFT, forwarded logs). State the gap explicitly.
- **Timezone mismatch between sources** → convert all to UTC before merging;
  never sort a mixed-tz timeline.

## Common pitfalls

- Answering from one log when the Sherlock rewards corroboration; single-source
  claims get contradicted by a later artifact.
- Timezone drift - the #1 cause of wrong "when" answers. Normalise to UTC once.
- Ignoring the requested answer format (Sherlocks are strict: ISO8601, exact
  filename casing, defanged vs plain). Re-read the question before submitting.
- Not re-pivoting: each new IOC (a hash, a filename, a PID) should be swept back
  across every source; investigations converge by iteration.
- Building the timeline in a spreadsheet with inconsistent timestamp formats -
  use Timeline Explorer or pandas with a parsed datetime.
- Chasing noise before anchoring - fix the anchor event first, then expand out.
- Forgetting the kill-chain gaps: if you have C2 but no initial access, keep
  digging; the story must be continuous.

## Proof / verification

- Every answer cites the artifact + exact record (Event ID/EventRecordID, log
  line number, MFT entry, pcap stream) and a UTC timestamp.
- Key pivots verified across >=2 independent sources (e.g. Sysmon 3 net conn +
  firewall log for the same C2 flow).
- The final narrative reads as an unbroken kill chain; note any evidence gaps and
  how you covered them.
- Re-answer each Sherlock question in its required format before submitting.

## Cross-references

- [sa--instructions--forensics-dfir--forensics-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--forensics-triage.md) - route each artifact in the bundle to its parser.
- [sa--instructions--forensics-dfir--windows-event-logs.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--windows-event-logs.md) - Event ID semantics behind the timeline rows.
- [sa--instructions--forensics-dfir--memory-volatility.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--memory-volatility.md) / [sa--instructions--forensics-dfir--disk-image-analysis.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--disk-image-analysis.md) - host-artifact deep dives.
- [sa--instructions--forensics-dfir--pcap-network-forensics.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--pcap-network-forensics.md) - the network slice of the timeline.
- [sa--instructions--forensics-dfir--soc-alert-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--soc-alert-triage.md) - if the bundle starts from a single alert.
- [sa--instructions--forensics-dfir--threat-hunting-detection.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--threat-hunting-detection.md) - convert findings into reusable detections.

## Credible sources

- HackTheBox Sherlocks (official): https://www.hackthebox.com/blog/introducing-sherlocks
- SANS DFIR incident timeline / "Hunt Evil" posters: https://www.sans.org/posters/hunt-evil/
- Plaso / log2timeline: https://plaso.readthedocs.io/en/latest/
- Eric Zimmerman Timeline Explorer & tools: https://ericzimmerman.github.io/
- Chainsaw: https://github.com/WithSecureLabs/chainsaw
- Hayabusa: https://github.com/Yamato-Security/hayabusa
- MITRE ATT&CK (kill chain mapping): https://attack.mitre.org/
- The DFIR Report (real-world intrusion timelines): https://thedfirreport.com/
