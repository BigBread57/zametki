# Threat Hunting & Detection Engineering

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/forensics-dfir/threat-hunting-detection.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/forensics-dfir/threat-hunting-detection.md)

---

---
name: threat-hunting-detection
category: forensics-dfir
triggers: ["threat hunting", "hypothesis-driven hunt", "sigma rule", "detection engineering", "ATT&CK hunt", "KQL", "Splunk SPL", "hunt query", "detection rule", "behavioral detection", "purple team"]
tools: ["Sigma / sigma-cli (pySigma)", "chainsaw", "hayabusa", "Splunk SPL", "Microsoft Sentinel/Defender KQL", "Elastic/EQL", "Zeek", "Velociraptor", "Jupyter/pandas", "MITRE ATT&CK Navigator"]
priority: medium
description: Hypothesis-driven threat hunting and detection engineering - form ATT&CK-based hypotheses, hunt with Sigma/KQL/SPL query patterns, and convert findings into durable detections.
---

# Threat Hunting & Detection Engineering

## Trigger context

The task is proactive, not alert-driven: "find evil in this dataset with no
alert", "write a detection for technique X", "hunt for lateral movement /
persistence / credential dumping", "convert this Sigma rule to Splunk/KQL". No
single artifact is flagged - you generate hypotheses from ATT&CK and known TTPs,
then test them against the available telemetry. Contrast with
[sa--instructions--forensics-dfir--soc-alert-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--soc-alert-triage.md) (reactive, one alert) and [sa--instructions--forensics-dfir--log-analysis-sherlock.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--log-analysis-sherlock.md)
(reconstruct a known incident).

## Objective

Run the hunt loop - hypothesis → data → query → analyse → conclude - to surface
malicious activity that evaded existing detections, and turn each confirmed
finding into a tested, low-false-positive detection rule (Sigma → SIEM query)
mapped to MITRE ATT&CK.

## Methodology (decision tree)

```
1. HYPOTHESIS   Pick an ATT&CK technique + a concrete assumption:
                "An adversary uses encoded PowerShell (T1059.001) for execution;
                 if present it appears as powershell -enc with Office/web parent."
2. DATA CHECK   Do we have the telemetry to test it? (Sysmon 1, 4104, EDR proc
                events, proxy, DNS). No data -> note the visibility gap, adjust.
3. QUERY        Express the hypothesis as a query over that telemetry
                (Sigma -> compile to SPL/KQL/EQL, or write native).
4. HUNT         Run broad, then reduce noise: stack/aggregate rare values,
                baseline normal, isolate outliers (long tail = suspicious).
5. ANALYSE      Investigate hits: TP (real evil), FP (benign), or new lead.
                Pivot found IOCs across other data (feeds sherlock workflow).
6. CONCLUDE     Hypothesis confirmed/refuted. Document either way.
7. OPERATIONALISE  Turn a TP pattern into a Sigma rule; tune thresholds; add to
                the SIEM; update the ATT&CK coverage map (Navigator).
```

## Commands & payloads

```bash
# --- Sigma: write once, compile to every backend (pySigma / sigma-cli) --------
pip install sigma-cli pysigma-backend-splunk pysigma-backend-elasticsearch \
            pysigma-backend-microsoft365defender
sigma convert -t splunk         -p sysmon rule.yml     # -> SPL
sigma convert -t elasticsearch  -p sysmon rule.yml     # -> Lucene/DSL
sigma convert -t kibana         rule.yml               # -> KQL (Kibana)
sigma convert -t microsoft365defender rule.yml         # -> Sentinel/MDE KQL
sigma list targets                                     # available backends

# Example Sigma rule (encoded PowerShell, T1059.001)
cat > enc_ps.yml <<'YAML'
title: Encoded PowerShell Execution
status: experimental
logsource: {product: windows, category: process_creation}
detection:
  sel:
    Image|endswith: '\powershell.exe'
    CommandLine|contains: ['-enc', '-EncodedCommand', ' -e ']
  condition: sel
level: high
tags: [attack.execution, attack.t1059.001]
YAML

# --- Hunt directly over EVTX with Sigma (no SIEM needed) ---------------------
chainsaw hunt ./evtx -s ./sigma-rules --mapping mappings/sigma-event-logs-all.yml --csv -o out
hayabusa csv-timeline -d ./evtx -o hunt.csv --RFC-3339

# --- Splunk SPL hunt patterns ------------------------------------------------
# Rare-process stacking (long tail = suspicious): least-frequent binaries
#   index=edr sourcetype=sysmon EventCode=1 | stats count by Image | sort count | head 25
# LOLBin remote fetch (T1105/T1218)
#   index=edr EventCode=1 (Image IN ("*certutil.exe","*bitsadmin.exe","*mshta.exe"))
#     CommandLine="*http*" | table _time host User Image CommandLine
# Suspicious parent->child (Office spawning a shell, T1059)
#   ParentImage IN ("*winword.exe","*excel.exe","*outlook.exe")
#     Image IN ("*cmd.exe","*powershell.exe","*wscript.exe")
# Beacon-like periodic DNS (T1071.004): stddev of intervals near zero
#   index=dns | streamstats current=f last(_time) as prev by query
#     | eval gap=_time-prev | stats stdev(gap) avg(gap) count by query dest

# --- Microsoft Sentinel / Defender KQL patterns ------------------------------
#   DeviceProcessEvents
#   | where FileName == "powershell.exe"
#   | where ProcessCommandLine has_any ("-enc","-EncodedCommand","FromBase64String")
#   | project Timestamp, DeviceName, AccountName, ProcessCommandLine, InitiatingProcessFileName
#
#   // LSASS access (credential dumping T1003.001)
#   DeviceEvents | where ActionType == "OpenProcessApiCall"
#   | where InitiatingProcessFileName !in~ ("MsMpEng.exe","taskmgr.exe")
#   | where FileName == "lsass.exe"
#
#   // New service persistence (T1543.003)
#   DeviceEvents | where ActionType == "ServiceInstalled"
#   | summarize count() by ServiceName, DeviceName, InitiatingProcessCommandLine

# --- Elastic EQL (sequence = ordered behaviour) ------------------------------
#   sequence by host.id with maxspan=1m
#     [process where process.parent.name : ("winword.exe","excel.exe")]
#     [process where process.name : ("powershell.exe","cmd.exe")]

# --- Coverage mapping --------------------------------------------------------
# Track which techniques your rules cover in MITRE ATT&CK Navigator (JSON layer).
```

## Branching logic

- **Have Sysmon/EDR process telemetry** → hunt execution/injection/LOLBins with
  process-creation queries; richest surface.
- **Only network telemetry (Zeek/proxy/DNS)** → hunt C2: beacon periodicity,
  rare JA3, long/high-entropy DNS, newly-registered domains, large outbound.
- **Hypothesis about persistence** → services (7045), tasks (4698), Run keys
  (Sysmon 13), WMI subscriptions; stack for rare/new entries.
- **Hypothesis about lateral movement** → 4624 type 3/10 graphs, admin shares,
  remote service creation, WinRM/WMI; look for one source → many targets.
- **A hunt confirms evil** → pivot IOCs into [sa--instructions--forensics-dfir--log-analysis-sherlock.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--log-analysis-sherlock.md) for full
  scoping, and operationalise the pattern as a Sigma rule.
- **No telemetry for the hypothesis** → record the visibility gap as a finding
  (a logging/deployment recommendation is a valid hunt outcome).

## Common pitfalls

- Hunting without a hypothesis - aimless querying rarely finds anything and
  can't be measured. State the technique and the expected artifact up front.
- Building a detection that only matches the one sample you saw (brittle IOCs)
  instead of the behaviour (durable TTP). Detect the technique, not the hash.
- Shipping a rule without FP testing against normal data - noisy rules get muted
  and miss the real thing. Baseline first.
- Mismatched Sigma pipeline: convert with the correct `-p` field-mapping for your
  data source (e.g. `sysmon`) or field names won't line up.
- Ignoring visibility gaps - a "no results" hunt over data you don't collect is
  not a clean bill of health.
- Not versioning/mapping detections to ATT&CK, so coverage and overlap are
  invisible.

## Proof / verification

- Each hunt documents: hypothesis, data sources, exact query, result
  (confirmed/refuted), and any findings - reproducible by another hunter.
- Confirmed findings map to an ATT&CK technique ID and carry evidence rows.
- New detections are tested for true positives (against the finding) and false
  positives (against baseline) before deployment; note the FP rate.
- ATT&CK Navigator layer updated to reflect new coverage.

## Cross-references

- [sa--instructions--forensics-dfir--windows-event-logs.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--windows-event-logs.md) - Event IDs and Sysmon fields the hunts query.
- [sa--instructions--forensics-dfir--soc-alert-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--soc-alert-triage.md) - detections here become the alerts triaged there.
- [sa--instructions--forensics-dfir--log-analysis-sherlock.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--log-analysis-sherlock.md) - scope out a hunt hit into a full investigation.
- [sa--instructions--forensics-dfir--pcap-network-forensics.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--pcap-network-forensics.md) - network-side hunting (beaconing, DNS tunneling).

## Credible sources

- Sigma / SigmaHQ (rules + spec): https://github.com/SigmaHQ/sigma
- Sigma documentation & backends: https://sigmahq.io/docs/
- sigma-cli (pySigma): https://github.com/SigmaHQ/sigma-cli
- MITRE ATT&CK: https://attack.mitre.org/
- MITRE ATT&CK Navigator: https://mitre-attack.github.io/attack-navigator/
- Splunk Security Content / detections: https://research.splunk.com/
- Microsoft KQL (Sentinel/Defender) reference: https://learn.microsoft.com/en-us/kusto/query/
- The ThreatHunting Project: https://www.threathunting.net/
- Elastic detection rules: https://github.com/elastic/detection-rules
