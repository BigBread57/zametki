# SOC Alert Triage

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/forensics-dfir/soc-alert-triage.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/forensics-dfir/soc-alert-triage.md)

---

---
name: soc-alert-triage
category: forensics-dfir
triggers: ["SOC", "alert triage", "SIEM alert", "true positive", "false positive", "MITRE ATT&CK mapping", "IOC reputation", "VirusTotal", "AbuseIPDB", "escalation", "containment", "phishing alert", "EDR alert", "enrichment"]
tools: ["VirusTotal", "AbuseIPDB", "URLScan.io", "Shodan", "OTX AlienVault", "Hybrid Analysis", "AnyRun", "MISP", "SIEM (Splunk/Sentinel/Elastic)", "Cortex/TheHive", "whois/dig"]
priority: high
description: SOC analyst alert-handling workflow - enrich, decide true vs false positive, map to MITRE ATT&CK, reputation-check IOCs, and make the escalate/contain call.
---

# SOC Alert Triage

## Trigger context

The input is a single alert or detection to adjudicate: an EDR/AV hit, a SIEM
correlation rule firing, a phishing report, an IDS/IPS signature, a suspicious
logon. The job is not deep forensics - it is a fast, defensible verdict:
false positive, true positive, or escalate-for-investigation, plus the immediate
containment decision. Deep-dive branches (memory/disk/pcap/logs) are invoked only
if triage confirms a real incident.

## Objective

For each alert, produce: (1) an enriched picture of what fired and why, (2) a
TP/FP/benign-true-positive verdict with reasoning, (3) an ATT&CK technique
mapping, (4) IOC reputation results, and (5) an action - close, escalate, or
contain - with the evidence to justify it in the ticket.

## Methodology (decision tree)

```
1. UNDERSTAND ALERT  What rule/detection fired? On what asset? Which user? What
                     is the raw evidence (process, cmdline, connection, email)?
2. SEVERITY x ASSET  Combine detection severity with asset criticality
                     (DC/exec laptop/crown-jewel server >> test VM).
3. ENRICH            Pull context: parent process, user's normal behaviour,
                     asset owner, recent related alerts, is this a known change?
4. IOC REPUTATION    Hash -> VirusTotal; IP -> AbuseIPDB/VT/Shodan;
                     domain/URL -> URLScan/VT/OTX; correlate across sources.
5. VERDICT
     FALSE POSITIVE   -> benign explanation (admin tool, known job, scanner)
     BENIGN TRUE POS  -> real but authorised (pentest, sanctioned admin)
     TRUE POSITIVE    -> malicious/unauthorised -> escalate
6. ATT&CK MAP        Assign tactic + technique ID; drives expected next steps.
7. DECISION          Close w/ notes | Escalate to IR | Contain now (isolate host,
                     disable account, block IOC) if active & high-impact.
8. DOCUMENT          Ticket: evidence, enrichment, verdict, ATT&CK, action, IOCs.
```

## Commands & payloads

```bash
# --- 4. IOC reputation (API examples; also usable via the web UIs) -----------
# File hash reputation (VirusTotal v3)
curl -s -H "x-apikey: $VT_API_KEY" \
  "https://www.virustotal.com/api/v3/files/$SHA256" \
  | jq '.data.attributes.last_analysis_stats, .data.attributes.popular_threat_classification'

# IP reputation (AbuseIPDB)
curl -s -G "https://api.abuseipdb.com/api/v2/check" \
  --data-urlencode "ipAddress=203.0.113.9" -d maxAgeInDays=90 \
  -H "Key: $ABUSEIPDB_KEY" -H "Accept: application/json" \
  | jq '.data | {abuseConfidenceScore, totalReports, isp, domain, countryCode}'

# IP reputation (VirusTotal)
curl -s -H "x-apikey: $VT_API_KEY" \
  "https://www.virustotal.com/api/v3/ip_addresses/203.0.113.9" \
  | jq '.data.attributes.last_analysis_stats'

# URL / domain
curl -s -H "x-apikey: $VT_API_KEY" \
  "https://www.virustotal.com/api/v3/domains/evil.example.com" \
  | jq '.data.attributes.last_analysis_stats, .data.attributes.categories'
# urlscan.io submit + result, OTX AlienVault pulse lookup, Shodan host lookup.

# Passive infra context
dig +short evil.example.com ; whois 203.0.113.9 | grep -Ei 'orgname|country|netname'

# --- 3. SIEM enrichment queries (adapt to your platform) ---------------------
# Splunk: pull the full context around the alerting event
#   index=edr host=WKS01 (process=powershell.exe) | table _time user parent_process command
# Sentinel/Defender KQL: has this hash/IP been seen elsewhere?
#   DeviceProcessEvents | where SHA256 == "<hash>" | project Timestamp,DeviceName,AccountName,ProcessCommandLine
#   DeviceNetworkEvents | where RemoteIP == "203.0.113.9"

# --- 7. Containment actions (examples) ---------------------------------------
# Defender: isolate device / MDE live response; Sentinel playbook.
# AD: Disable-ADAccount -Identity compromised.user
# Firewall/EDR: block the C2 IP/domain and the file hash across the fleet.
```

## Triage matrix (verdict heuristics)

```
Strong TRUE POSITIVE signals:
  - Hash flagged by many AV engines / known malware family (VT).
  - IP high AbuseIPDB confidence + your host initiated outbound to it.
  - LOLBin abuse: rundll32/regsvr32/mshta/certutil fetching remote payloads.
  - Encoded PowerShell (-enc), parent = Office/browser spawning a shell.
  - LSASS access by a non-security tool; new service = PsExec pattern.
Likely FALSE POSITIVE / benign:
  - Signed vendor binary from expected path; matches a known change window.
  - Internal vuln scanner / admin jump host as the "source".
  - Clean reputation across VT/AbuseIPDB/OTX + behaviour fits user baseline.
Escalate regardless when: crown-jewel asset, active outbound C2, credential
  access, or ransomware-like mass file writes.
```

## Branching logic

- **Clean reputation + benign context + matches a change** → close as FP with a
  note; consider a tuning suggestion for the rule.
- **Malicious reputation OR clearly malicious behaviour on a real asset** → true
  positive → escalate to [sa--instructions--forensics-dfir--log-analysis-sherlock.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--log-analysis-sherlock.md) for full-scope investigation.
- **Active C2 / lateral movement in progress** → contain first (isolate host /
  disable account / block IOC), then investigate - do not wait.
- **Phishing report** → detonate URL/attachment in a sandbox (AnyRun/Hybrid
  Analysis/urlscan), pull VT verdict, find other recipients, block sender/URL.
- **Ambiguous** → widen enrichment (more SIEM history, ask asset owner) rather
  than guessing; a documented "insufficient evidence, monitoring" is valid.
- **Confirmed malware sample** → hand the file to [sa--instructions--forensics-dfir--forensics-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--forensics-triage.md); the C2
  from sandboxing feeds [sa--instructions--forensics-dfir--pcap-network-forensics.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--pcap-network-forensics.md) and the block list.

## Common pitfalls

- Verdict from a single source. Correlate VT + AbuseIPDB + OTX + internal
  context; any one can be wrong or stale.
- Uploading sensitive/internal samples to public VT (leaks data + tips off the
  adversary). Search by hash first; detonate in a private sandbox if needed.
- Ignoring asset criticality - the same alert on a DC vs a test VM is a different
  severity.
- Alert fatigue closing real TPs as FP without checking behaviour, not just
  reputation (novel malware has clean hashes).
- Skipping the ATT&CK mapping - it drives what to look for next and how to
  contain.
- Not preserving evidence before containment (grab volatile data if you isolate).
- Failing to document reasoning; the ticket must let the next analyst reproduce
  the verdict.

## Proof / verification

- Verdict is backed by named enrichment: engine counts, confidence scores,
  behavioural evidence, and asset context.
- ATT&CK technique ID assigned (e.g. T1059.001 encoded PowerShell, T1003.001
  LSASS dumping, T1566 phishing).
- IOCs listed (defanged) with their reputation verdicts.
- Action recorded with justification; if contained, note what and when.

## Cross-references

- [sa--instructions--forensics-dfir--log-analysis-sherlock.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--log-analysis-sherlock.md) - full investigation when triage escalates.
- [sa--instructions--forensics-dfir--windows-event-logs.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--windows-event-logs.md) - the Event IDs behind endpoint alerts.
- [sa--instructions--forensics-dfir--pcap-network-forensics.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--pcap-network-forensics.md) - validate/scope network IOCs.
- [sa--instructions--forensics-dfir--threat-hunting-detection.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--threat-hunting-detection.md) - turn a confirmed TP into a broader hunt/rule.
- [sa--instructions--forensics-dfir--forensics-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--forensics-triage.md) - handle any malware sample attached to the alert.

## Credible sources

- MITRE ATT&CK: https://attack.mitre.org/
- VirusTotal API v3 docs: https://docs.virustotal.com/reference/overview
- AbuseIPDB API: https://docs.abuseipdb.com/
- urlscan.io: https://urlscan.io/docs/api/
- AlienVault OTX: https://otx.alienvault.com/api
- Shodan: https://developer.shodan.io/
- MISP threat intel platform: https://www.misp-project.org/
- SANS SOC / incident handling: https://www.sans.org/posters/?focus-area=cyber-defense
- CISA incident response guidance: https://www.cisa.gov/resources-tools/services/cisa-incident-response-training
