# Global CTF Operating Loop

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/_meta/00-global-methodology.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/_meta/00-global-methodology.md)

---

---
name: global-ctf-methodology
category: _meta
triggers: [ctf, start, new box, new challenge, "where do i start", target, engagement, foothold]
tools: [nmap, ffuf, netcat, tmux]
priority: high
description: Top-level operating loop an AI agent runs on ANY CTF target before dispatching to a category playbook.
---

# Global CTF Operating Loop

## Trigger context
Load this FIRST on any new target, box, or challenge file before choosing a specialised playbook.

## The core loop (OODA for CTF)
1. **Classify the challenge type.** Is it a *jeopardy* task (a file/URL/blob in a category) or a *host/network* target (IP, box)? This decides which category folder you route into.
2. **Enumerate broadly, then narrow.** Never exploit before you have mapped the attack surface. Breadth-first: what services/ports/endpoints/functions exist? Then depth-first on the most promising.
3. **One hypothesis at a time.** Form a specific, falsifiable hypothesis ("this `id` param is SQLi"). Test the cheapest disproof first. Kill dead hypotheses fast — CTF failure mode #1 is rabbit-holing.
4. **Log everything.** Keep a running notes file: every credential, hash, path, version, and endpoint. Credentials and version strings are the connective tissue between stages.
5. **Escalate the loop.** Foothold → local enum → privesc → loot → pivot. Re-run enumeration after every privilege change; new access reveals new surface.
6. **Capture with proof.** A flag is only found when you can display it. Don't report success on inference.

## Routing table (which folder to load)
| Signal | Route to |
|---|---|
| HTTP/HTTPS, web app, API, cookies, params | `web/` |
| Linux shell obtained, need root | `host-linux/` |
| Windows shell obtained, need SYSTEM/admin | `host-windows/` |
| Domain, DC, Kerberos, LDAP, SMB, `.local` | `active-directory/` |
| SSH creds/keys, need to pivot/tunnel | `access-pivoting/` |
| Binary given, understand behaviour | `reverse-engineering/` |
| Binary given, exploit it for a shell | `binary-exploitation/` |
| Ciphertext, keys, math, encoding | `crypto/` |
| pcap, memory dump, disk image, logs, "Sherlock" | `forensics-dfir/` |
| Suspicious sample, IOC hunt, unpack | `malware-analysis/` |
| Image/audio with hidden data, OSINT, jail escape | `stego-osint-misc/` |

## Universal first moves for a network target
```bash
# Fast full-port sweep, then targeted service/version scan
nmap -p- --min-rate 5000 -T4 -oN ports.txt <IP>
nmap -sCV -p <open_ports> -oN services.txt <IP>
# Note every version string; search each for known exploits.
```

## Time-boxing rule
If a hypothesis has burned your budget without a signal of progress, drop it and return to the enumeration notes for the next-most-promising lead. Document why you abandoned it.

## Cross-references
- Web entry: [web-recon](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--web-recon.md) · Linux root: [linux-enum-linpeas](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--linux-enum-linpeas.md) · Windows root: [windows-enum-winpeas](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--windows-enum-winpeas.md) · AD: [ad-enum-bloodhound](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--ad-enum-bloodhound.md)

## Credible sources
- HackTricks — https://book.hacktricks.wiki/
- The TCM/OSCP-style enumeration methodology — https://tcm-sec.com/
