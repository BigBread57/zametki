# CTF Methodology Skill — Agent Router

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / SKILL.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/SKILL.md)

---

---
name: ctf-methodology
description: >-
  Master playbook router for CTF and authorized offensive-security work. Invoke for any CTF challenge, HackTheBox/TryHackMe box, or pentest lab across: web exploitation (SQLi, XSS, SSTI, SSRF, LFI, XXE, deserialization, upload, JWT/auth, IDOR, API/GraphQL/DRF, NoSQLi, CMS-to-CVE); Linux/Windows privilege escalation (linpeas, winpeas, sudo/GTFOBins, SUID, tokens/Potato); Active Directory (BloodHound, Kerberoasting, AS-REP, NTLM relay, DCSync, ADCS); SSH access & pivoting; reverse engineering (Ghidra, GDB, angr); binary exploitation/pwn (ROP, heap, format string); cryptography (RSA, AES, XOR, hashes); forensics/DFIR & SOC (Volatility, pcap, EVTX, Sherlock, MITRE ATT&CK, threat hunting); malware analysis; hardware/firmware (UART, JTAG, SPI flash, SDR, glitching); steganography; and OSINT. Routes to the exact playbook by matching trigger keywords, scan output, file types, and error signatures. Authorized CTF/lab use only.
license: For authorized security testing, CTF, and education only.
---

# CTF Methodology Skill — Agent Router

A trigger-routed library of **methodology-only** playbooks for autonomous CTF / authorized
offensive-security work. Each playbook is a self-contained decision tree: *when to fire, what
to do in order, exact commands, how to branch, how to prove the flag.*

> **Authorization guardrail.** Use only against targets you are explicitly authorized to test:
> CTF platforms, personal labs, or engagements with written scope. These techniques are
> offensive; do not run them against systems you do not own or lack permission to test.

## How to use this skill
1. **On every new target, load [sa--instructions--_meta--00-global-methodology.md](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/sa--instructions--_meta--00-global-methodology.md) first.** It classifies
   the challenge (jeopardy category vs. host/network) and dispatches to a category below.
2. **Match signals to triggers.** Compare the challenge prompt, `nmap`/scan output, file types,
   obtained access level, and error strings against each file's `triggers:` (case-insensitive
   keyword / substring match; quoted phrases are exact signals like error text or tool names).
3. **Load the matched playbook(s)**, highest `priority` first, and follow its **Methodology**
   section in order. Files live at the `Path:` shown in the index below (relative to this skill).
4. **Re-enumerate after every privilege change** — new access reveals new surface.
5. **Prove before claiming success** — a flag is found only when you can display it.

## Category routing table
| Signal | Category folder |
|---|---|
| HTTP/HTTPS, web app, API, cookies, params, CMS | `instructions/web/` |
| Linux shell obtained, need root | `instructions/host-linux/` |
| Windows shell obtained, need SYSTEM/admin | `instructions/host-windows/` |
| Domain, DC, Kerberos, LDAP, SMB, `.local` | `instructions/active-directory/` |
| SSH creds/keys, need to pivot/tunnel | `instructions/access-pivoting/` |
| Binary given, understand behaviour | `instructions/reverse-engineering/` |
| Binary given, exploit for a shell | `instructions/binary-exploitation/` |
| Ciphertext, keys, math, encoding | `instructions/crypto/` |
| pcap, memory dump, disk image, logs, "Sherlock", SIEM, alert | `instructions/forensics-dfir/` |
| Suspicious sample, IOC hunt, unpack, macro | `instructions/malware-analysis/` |
| Firmware, PCB, UART/JTAG, SPI flash, SDR, glitch | `instructions/hardware/` |
| Hidden data in media, OSINT, jail escape | `instructions/stego-osint-misc/` |

## Trigger routing rules
- Triggers are matched **case-insensitively** as keywords/substrings.
- Quoted multi-word triggers (e.g. `"union select"`, `"SeImpersonatePrivilege"`, `"169.254.169.254"`)
  are high-confidence exact signals — prefer them.
- If multiple files match, load by `priority` (high → low), then by most specific trigger.
- If nothing matches, fall back to [sa--instructions--_meta--00-global-methodology.md](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/sa--instructions--_meta--00-global-methodology.md) and re-route.

---

# Complete playbook index (auto-generated — paths & triggers are exact)

### _meta

- **`global-ctf-methodology`** — _priority: high_
  - Path: [sa--instructions--_meta--00-global-methodology.md](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/sa--instructions--_meta--00-global-methodology.md)
  - Triggers: `[ctf, start, new box, new challenge, "where do i start", target, engagement, foothold]`
  - Top-level operating loop an AI agent runs on ANY CTF target before dispatching to a category playbook.

### web

- **`api-graphql-rest`** — _priority: high_
  - Path: [sa--instructions--web--api-graphql-rest.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--api-graphql-rest.md)
  - Triggers: `[graphql, "/graphql", introspection, "__schema", swagger, openapi, "/api/", rest, "wsdl", drf, "django-rest", browsable-api, "format=json", "?format=", batching]`
  - Discover and exploit REST/GraphQL APIs - introspection, batching, field suggestion, OpenAPI/Swagger discovery, and Django REST Framework weaknesses.
- **`auth-jwt-session`** — _priority: high_
  - Path: [sa--instructions--web--auth-jwt-session.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--auth-jwt-session.md)
  - Triggers: `[jwt, jws, "eyJ", authorization-bearer, session-cookie, "alg:none", "HS256", "RS256", jwks, kid, oauth, saml, password-reset, 2fa, otp, session-fixation, remember-me]`
  - Attack JWTs, session tokens, and auth/OAuth/2FA flows to forge identity, escalate privilege, or bypass login.
- **`cms-and-known-exploits`** — _priority: high_
  - Path: [sa--instructions--web--cms-and-known-exploits.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--cms-and-known-exploits.md)
  - Triggers: `[cms, wordpress, drupal, joomla, magento, tomcat, jenkins, phpmyadmin, wpscan, droopescan, joomscan, cmseek, nuclei, searchsploit, "known exploit", cve, "version disclosure", "CHANGELOG.txt", generator, "X-Powered-By", "default credentials", "n-day"]`
  - Fingerprint the stack/CMS, pin the exact version, then map that version to known CVEs/exploits and fire them.
- **`command-injection`** — _priority: high_
  - Path: [sa--instructions--web--command-injection.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--command-injection.md)
  - Triggers: `[rce, "command injection", "os command", "shell injection", "ping", "nslookup", "argument injection", ";id", "|whoami", "$(", backtick, "blind command injection", collaborator, "oob dns"]`
  - Detect and exploit OS command injection — direct, blind, and out-of-band — plus filter/space bypass and reverse shells.
- **`file-inclusion-lfi-rfi`** — _priority: high_
  - Path: [sa--instructions--web--file-inclusion-lfi-rfi.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--file-inclusion-lfi-rfi.md)
  - Triggers: `[lfi, rfi, "local file inclusion", "remote file inclusion", "path traversal", "directory traversal", "../", "php://filter", "php://", "data://", "expect://", phar, "log poisoning", "/etc/passwd", "proc/self/environ", "?page=", "?file=", "?include=", wrapper, "filter chain"]`
  - Read arbitrary files via traversal/wrappers and escalate LFI to RCE through log poisoning, PHP filter chains, wrappers, and phar.
- **`file-upload`** — _priority: high_
  - Path: [sa--instructions--web--file-upload.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--file-upload.md)
  - Triggers: `[upload, "file upload", "profile picture", avatar, "image upload", multipart, "content-type", "magic bytes", webshell, "web shell", polyglot, ".htaccess", "double extension", "shell.php", "upload avatar", attachment, imagetragick, "svg upload"]`
  - Bypass upload filters (extension/MIME/magic byte) to plant web shells, polyglots, and pivot via path traversal and race conditions.
- **`idor-access-control`** — _priority: high_
  - Path: [sa--instructions--web--idor-access-control.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--idor-access-control.md)
  - Triggers: `[idor, "id=", uuid, "/users/", "/api/", object-reference, mass-assignment, forced-browsing, "admin", privilege-escalation, verb-tampering, parameter-tampering, bola]`
  - Find and exploit broken access control - IDOR/BOLA, privilege escalation, mass assignment, forced browsing, and HTTP verb tampering.
- **`insecure-deserialization`** — _priority: high_
  - Path: [sa--instructions--web--insecure-deserialization.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--insecure-deserialization.md)
  - Triggers: `[deserialization, unserialize, pickle, ysoserial, "ysoserial.net", phpggc, phar, "gadget chain", viewstate, "__VIEWSTATE", "rO0AB", "aced0005", "O:8:", "__wakeup", "__destruct", marshal, "readObject", "BinaryFormatter", "pickle.loads", "yaml.load", serialized]`
  - Achieve RCE/file access by tampering with serialized objects across PHP, Python, Java, .NET, and Ruby using magic methods and gadget chains.
- **`nosql-injection`** — _priority: high_
  - Path: [sa--instructions--web--nosql-injection.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--nosql-injection.md)
  - Triggers: `[mongodb, mongoose, nosql, "$ne", "$gt", "$regex", "$where", "$in", json-login, "username[$ne]", couchdb, firebase, bson]`
  - Exploit NoSQL (MongoDB) operator injection for authentication bypass, boolean/blind data extraction, and server-side JS execution.
- **`sql-injection`** — _priority: high_
  - Path: [sa--instructions--web--sql-injection.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--sql-injection.md)
  - Triggers: `[sqli, "sql injection", sqlmap, union, "boolean blind", "time-based", "error-based", "syntax error", "You have an error in your SQL syntax", ORA-, "quoted string not properly terminated", "unclosed quotation mark", "?id=", "order by", "sqlite", mysql, postgres, mssql]`
  - Detect and exploit SQL injection manually and with sqlmap — union, blind boolean/time, error-based, plus WAF bypass.
- **`ssrf`** — _priority: high_
  - Path: [sa--instructions--web--ssrf.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--ssrf.md)
  - Triggers: `[ssrf, "server side request forgery", "server-side request forgery", url, "fetch url", webhook, "image from url", "pdf generator", proxy, redirect, "169.254.169.254", metadata, gopher, "internal port scan", "blind ssrf", curl, "url=", "check url"]`
  - Coerce the server into making attacker-controlled requests to internal/cloud targets for file read, metadata theft, port scan, and RCE.
- **`ssti`** — _priority: high_
  - Path: [sa--instructions--web--ssti.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--ssti.md)
  - Triggers: `[ssti, "template injection", "server-side template", jinja2, twig, freemarker, velocity, mako, smarty, handlebars, "{{7*7}}", "49", "${7*7}", "TemplateSyntaxError", "name=", "greeting", "email template"]`
  - Identify the template engine via probes, then chain to RCE — Jinja2/Twig/Freemarker/Velocity/Mako/Smarty polyglots.
- **`web-recon`** — _priority: high_
  - Path: [sa--instructions--web--web-recon.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--web-recon.md)
  - Triggers: `[recon, enumeration, "content discovery", "directory brute force", ffuf, gobuster, dirsearch, feroxbuster, "vhost fuzzing", "subdomain enumeration", "parameter discovery", arjun, "tech fingerprint", whatweb, "404 page", "hidden endpoint"]`
  - First-move web enumeration: fingerprint stack, brute force content, fuzz vhosts/subdomains, discover hidden params.
- **`xss`** — _priority: medium_
  - Path: [sa--instructions--web--xss.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--xss.md)
  - Triggers: `[xss, "cross-site scripting", reflected, stored, "dom xss", "innerHTML", "document.write", "location.hash", csp, "content-security-policy", "alert(1)", "cookie theft", "javascript:", onerror, "search=", comment]`
  - Find and weaponize reflected/stored/DOM XSS — context-aware payloads, CSP bypass, cookie/session exfiltration.
- **`xxe`** — _priority: high_
  - Path: [sa--instructions--web--xxe.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--xxe.md)
  - Triggers: `[xxe, "xml external entity", xml, "<!DOCTYPE", "<!ENTITY", soap, svg, docx, xlsx, "content-type: application/xml", "content-type: text/xml", "parameter entity", "external dtd", "billion laughs", oob]`
  - Abuse XML parsers to read local files, perform SSRF, and exfiltrate data out-of-band via external/parameter entities and malicious DTDs.

### host-linux

- **`capabilities`** — _priority: high_
  - Path: [sa--instructions--host-linux--capabilities.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--capabilities.md)
  - Triggers: `["getcap", "cap_setuid", "cap_dac_read_search", "cap_dac_override", "capabilities", "cap_chown", "cap_sys_admin", "cap_setgid", "ep permitted effective"]`
  - Enumerate Linux file capabilities and exploit cap_setuid, cap_dac_read_search, cap_dac_override and friends for privilege escalation.
- **`cron-path-hijack`** — _priority: high_
  - Path: [sa--instructions--host-linux--cron-path-hijack.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--cron-path-hijack.md)
  - Triggers: `["writable cron", "crontab", "cron.d", "wildcard injection", "tar wildcard", "PATH hijack", "writable PATH dir", "pspy root job", "scheduled task"]`
  - Escalate via root cron jobs — writable scripts, PATH hijacking, and wildcard (tar/rsync/chown) injection.
- **`kernel-and-misc-privesc`** — _priority: high_
  - Path: [sa--instructions--host-linux--kernel-and-misc-privesc.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--kernel-and-misc-privesc.md)
  - Triggers: `["uname -a", "old kernel", "DirtyCow", "DirtyPipe", "PwnKit", "pkexec", "docker group", "lxd group", "NFS no_root_squash", "writable /etc/passwd", "python library hijack", "kernel exploit"]`
  - Kernel-version-to-exploit mapping plus high-value misconfigs — pkexec, docker/lxd groups, NFS no_root_squash, writable /etc/passwd, library hijacking.
- **`linux-enum-linpeas`** — _priority: high_
  - Path: [sa--instructions--host-linux--linux-enum-linpeas.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--linux-enum-linpeas.md)
  - Triggers: `["low-priv shell", "linpeas", "LinEnum", "pspy", "initial enumeration", "privesc enumeration", "what can I do as this user", "www-data shell"]`
  - First-move enumeration on a fresh low-priv Linux shell using automated tools plus a manual checklist, and how to triage the output.
- **`sudo-gtfobins`** — _priority: high_
  - Path: [sa--instructions--host-linux--sudo-gtfobins.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--sudo-gtfobins.md)
  - Triggers: `["sudo -l", "NOPASSWD", "GTFOBins", "sudo allowed command", "env_keep", "LD_PRELOAD", "Baron Samedit", "sudoedit", "sudo version"]`
  - Turn sudo -l output into a root shell via GTFOBins escapes, env/LD_PRELOAD abuse, or a sudo CVE.
- **`suid-sgid`** — _priority: high_
  - Path: [sa--instructions--host-linux--suid-sgid.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--suid-sgid.md)
  - Triggers: `["SUID binary", "SGID binary", "perm -4000", "find suid", "GTFOBins SUID", "shared object injection", "PATH abuse suid"]`
  - Enumerate and exploit set-uid/set-gid binaries via GTFOBins escapes, shared-object injection, or PATH abuse.

### host-windows

- **`credential-hunting`** — _priority: high_
  - Path: [sa--instructions--host-windows--credential-hunting.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--credential-hunting.md)
  - Triggers: `["SAM", "SYSTEM hive", "LSASS", "mimikatz", "autologon", "unattend.xml", "cmdkey", "DPAPI", "wifi password", "saved credentials", "findstr password", "secretsdump"]`
  - Hunt stored Windows credentials - SAM/SYSTEM hives, LSASS, autologon and unattend files, saved creds, DPAPI, and Wi-Fi keys - to escalate or move laterally.
- **`service-misconfigurations`** — _priority: high_
  - Path: [sa--instructions--host-windows--service-misconfigurations.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--service-misconfigurations.md)
  - Triggers: `["unquoted service path", "insecure service permissions", "weak binary permissions", "DLL hijacking", "writable service registry", "sc config", "binPath", "accesschk", "SERVICE_CHANGE_CONFIG"]`
  - Enumerate and exploit misconfigured Windows services - insecure permissions, unquoted paths, weak binary/registry ACLs, and DLL hijacking - to run code as SYSTEM.
- **`token-impersonation-potato`** — _priority: high_
  - Path: [sa--instructions--host-windows--token-impersonation-potato.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--token-impersonation-potato.md)
  - Triggers: `["SeImpersonatePrivilege", "SeAssignPrimaryTokenPrivilege", "JuicyPotato", "PrintSpoofer", "GodPotato", "RoguePotato", "SweetPotato", "SeBackupPrivilege", "SeRestorePrivilege", "SeTakeOwnershipPrivilege", "SeDebugPrivilege", "potato"]`
  - Escalate to SYSTEM by abusing held Windows privileges - the Potato family for SeImpersonate/SeAssignPrimaryToken, plus SeBackup/SeRestore/SeTakeOwnership/SeDebug abuse.
- **`uac-and-registry`** — _priority: high_
  - Path: [sa--instructions--host-windows--uac-and-registry.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--uac-and-registry.md)
  - Triggers: `["AlwaysInstallElevated", "UAC bypass", "fodhelper", "autoruns", "scheduled tasks", "writable PATH", "msiexec elevated", "registry hijack", "startup folder", "schtasks"]`
  - Escalate via AlwaysInstallElevated, UAC bypasses (fodhelper), and hijackable autoruns, scheduled tasks, and writable PATH/binary directories.
- **`windows-enum-winpeas`** — _priority: high_
  - Path: [sa--instructions--host-windows--windows-enum-winpeas.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--windows-enum-winpeas.md)
  - Triggers: `["winpeas", "PowerUp", "Seatbelt", "whoami /priv", "whoami /all", "systeminfo", "windows enumeration", "privesc enum", "low-priv shell"]`
  - First-move Windows privilege-escalation enumeration - run automated scanners plus manual triage and decide which vector to pursue.

### active-directory

- **`ad-enum-bloodhound`** — _priority: high_
  - Path: [sa--instructions--active-directory--ad-enum-bloodhound.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--ad-enum-bloodhound.md)
  - Triggers: `["bloodhound", "sharphound", "bloodhound-python", "AD enumeration", "no creds", "null session", "RID cycling", "rid-brute", "ldap enum", "nxc smb", "attack path", "domain recon"]`
  - Initial Active Directory enumeration from zero access to first credentials, then BloodHound collection and attack-path analysis.
- **`adcs-esc`** — _priority: high_
  - Path: [sa--instructions--active-directory--adcs-esc.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--adcs-esc.md)
  - Triggers: `["adcs", "certipy", "ESC1", "ESC2", "ESC3", "ESC4", "ESC8", "certificate template", "pass the cert", "UnPAC the hash", "shadow credentials", "PKINIT", "web enrollment", "certfnsh"]`
  - Enumerate and exploit Active Directory Certificate Services misconfigurations (ESC1-ESC8) to obtain certificates that authenticate as privileged users.
- **`asreproasting`** — _priority: high_
  - Path: [sa--instructions--active-directory--asreproasting.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--asreproasting.md)
  - Triggers: `["asreproast", "AS-REP", "AS-REP roasting", "GetNPUsers", "DONT_REQ_PREAUTH", "no preauth", "hashcat 18200", "krb5asrep", "preauth disabled"]`
  - Recover crackable AS-REP hashes from accounts that have Kerberos pre-authentication disabled, often with no credentials required.
- **`dcsync-dump`** — _priority: high_
  - Path: [sa--instructions--active-directory--dcsync-dump.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--dcsync-dump.md)
  - Triggers: `["dcsync", "secretsdump", "ntds", "ntds.dit", "DRSUAPI", "GetChanges", "GetChangesAll", "krbtgt", "dump hashes", "replication rights", "domain dump"]`
  - Abuse directory-replication rights (or DC admin) to pull all domain credential material, including krbtgt, via DCSync or NTDS.dit dumping.
- **`kerberoasting`** — _priority: high_
  - Path: [sa--instructions--active-directory--kerberoasting.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--kerberoasting.md)
  - Triggers: `["kerberoast", "kerberoasting", "GetUserSPNs", "SPN", "servicePrincipalName", "TGS-REP", "hashcat 13100", "krb5tgs", "Rubeus kerberoast", "targeted kerberoast"]`
  - Request TGS tickets for SPN-bearing accounts and crack them offline to recover service-account plaintext passwords.
- **`ntlm-relay-coerce`** — _priority: high_
  - Path: [sa--instructions--active-directory--ntlm-relay-coerce.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--ntlm-relay-coerce.md)
  - Triggers: `["ntlm relay", "ntlmrelayx", "responder", "PetitPotam", "coercer", "printerbug", "dfscoerce", "SMB signing", "RBCD", "ESC8", "relay to LDAP", "relay to ADCS", "CVE-2019-1040", "poisoning"]`
  - Capture or coerce NTLM authentication and relay it to SMB, LDAP, or ADCS to gain code execution, RBCD, or a certificate for privilege escalation.
- **`pth-ptt-lateral`** — _priority: high_
  - Path: [sa--instructions--active-directory--pth-ptt-lateral.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--pth-ptt-lateral.md)
  - Triggers: `["pass the hash", "pth", "pass the ticket", "ptt", "overpass the hash", "pass the key", "golden ticket", "silver ticket", "wmiexec", "psexec", "evil-winrm", "ccache", "KRB5CCNAME", "S4U", "lateral movement"]`
  - Reuse recovered NT hashes and Kerberos tickets to move laterally (PtH, OPtH, PtT) and forge Golden/Silver/S4U tickets for persistence and privilege escalation.

### access-pivoting

- **`pivoting-proxychains`** — _priority: high_
  - Path: [sa--instructions--access-pivoting--pivoting-proxychains.md](https://raw.githubusercontent.com/BigBread57/zametki/main/07-access-pivoting/sa--instructions--access-pivoting--pivoting-proxychains.md)
  - Triggers: `["proxychains", "chisel", "ligolo-ng", "ligolo", "pivot", "internal network", "double pivot", "socks5", "dead drop", "no ssh server", "subnet discovery"]`
  - Route tooling through a compromised host into internal networks using proxychains, chisel, and ligolo-ng, including double pivots.
- **`reverse-shells-and-callbacks`** — _priority: high_
  - Path: [sa--instructions--access-pivoting--reverse-shells-and-callbacks.md](https://raw.githubusercontent.com/BigBread57/zametki/main/07-access-pivoting/sa--instructions--access-pivoting--reverse-shells-and-callbacks.md)
  - Triggers: `[reverse shell, bind shell, "nc -lvnp", "nc -e", netcat, ncat, socat, "4444", callback, listener, "bash -i", "/dev/tcp", pwncat, "shell stabilization", pty, exfiltrate, exfil, "data exfiltration", oob, "out-of-band", egress, "curl attacker", interactsh, "python -m http.server", "did not connect back", "no callback", "get a shell", rce]`
  - Get a reverse/bind shell or out-of-band data callback from a target, choosing a callback port that survives egress filtering (reuse the target's open ports), with a fallback path when direct callbacks are blocked.
- **`ssh-access-and-keys`** — _priority: high_
  - Path: [sa--instructions--access-pivoting--ssh-access-and-keys.md](https://raw.githubusercontent.com/BigBread57/zametki/main/07-access-pivoting/sa--instructions--access-pivoting--ssh-access-and-keys.md)
  - Triggers: `["ssh", "id_rsa", "id_ed25519", "port 22", "authorized_keys", "known_hosts", "ssh-agent", "SSH_AUTH_SOCK", "ProxyJump", ".ssh", "private key", "passphrase"]`
  - Turn discovered SSH private keys, agent sockets, and writable authorized_keys into interactive shells, persistence, and lateral movement.
- **`tunneling-port-forwarding`** — _priority: high_
  - Path: [sa--instructions--access-pivoting--tunneling-port-forwarding.md](https://raw.githubusercontent.com/BigBread57/zametki/main/07-access-pivoting/sa--instructions--access-pivoting--tunneling-port-forwarding.md)
  - Triggers: `["port forward", "ssh -L", "ssh -R", "ssh -D", "SOCKS proxy", "tunnel", "internal service", "localhost only", "bind 127.0.0.1", "reverse tunnel", "sshuttle"]`
  - Reach services bound to an internal interface by building SSH local, remote, and dynamic (SOCKS) tunnels from a foothold.

### reverse-engineering

- **`angr-symbolic-execution`** — _priority: high_
  - Path: [sa--instructions--reverse-engineering--angr-symbolic-execution.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--angr-symbolic-execution.md)
  - Triggers: `["angr", "symbolic execution", "constraint solving", "flag checker", "explore find avoid", "z3", "SMT", "brute force alternative", "license check", "keygen", "deterministic checker"]`
  - Use angr symbolic execution to auto-solve deterministic input->accept flag checkers via explore(find/avoid), hooking, and constraint solving.
- **`anti-debug-and-packing`** — _priority: high_
  - Path: [sa--instructions--reverse-engineering--anti-debug-and-packing.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--anti-debug-and-packing.md)
  - Triggers: `["packed", "UPX", "high entropy", "anti-debug", "ptrace", "PTRACE_TRACEME", "detect debugger", "timing check", "obfuscation", "unpack", "manual unpack", "LD_PRELOAD", "segfault under gdb"]`
  - Detect and defeat packers (UPX/custom) and anti-debug tricks (ptrace, timing, /proc checks) to reach analyzable code.
- **`ghidra-workflow`** — _priority: high_
  - Path: [sa--instructions--reverse-engineering--ghidra-workflow.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--ghidra-workflow.md)
  - Triggers: `["ghidra", "decompiler", "decompile", "pseudo-C", "rename variable", "retype", "find main", "strcmp", "flag logic", "analyzeHeadless", "ghidra script", "stripped binary"]`
  - Import, auto-analyze, and read a binary in Ghidra's decompiler to follow flag logic, plus rename/retype and headless scripting.
- **`re-dynamic-gdb`** — _priority: high_
  - Path: [sa--instructions--reverse-engineering--re-dynamic-gdb.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--re-dynamic-gdb.md)
  - Triggers: `["gdb", "pwndbg", "GEF", "breakpoint", "dynamic analysis", "runtime", "patch comparison", "bypass check", "ltrace", "strace", "examine memory", "license check", "password check", "step through"]`
  - Drive a binary at runtime under gdb/pwndbg/GEF to watch comparisons, read secrets from memory, and patch or bypass checks live.
- **`re-triage-static`** — _priority: high_
  - Path: [sa--instructions--reverse-engineering--re-triage-static.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--re-triage-static.md)
  - Triggers: `["ELF binary", "PE binary", "Mach-O", "strings", "unknown binary", "file command", "checksec", "detect language", "detect packer", "first look", "quick win", "readelf", "objdump", "nm"]`
  - First-pass static triage of an unknown binary to fingerprint it, harvest low-hanging flags, and decide the next stage.

### binary-exploitation

- **`format-string`** — _priority: high_
  - Path: [sa--instructions--binary-exploitation--format-string.md](https://raw.githubusercontent.com/BigBread57/zametki/main/12-pwn/sa--instructions--binary-exploitation--format-string.md)
  - Triggers: `["format string", "printf", "%p", "%n", "%x", "fmtstr_payload", "GOT overwrite", "leak canary", "leak libc", "arbitrary write", "%hhn", "position offset"]`
  - Exploit user-controlled printf — leak stack/canary/libc with %p and gain arbitrary write with %n / fmtstr_payload.
- **`heap-exploitation`** — _priority: high_
  - Path: [sa--instructions--binary-exploitation--heap-exploitation.md](https://raw.githubusercontent.com/BigBread57/zametki/main/12-pwn/sa--instructions--binary-exploitation--heap-exploitation.md)
  - Triggers: `["heap", "malloc", "free", "tcache", "tcache poisoning", "double free", "use after free", "UAF", "fastbin dup", "__free_hook", "__malloc_hook", "safe linking", "how2heap", "chunk"]`
  - glibc malloc internals and modern heap primitives — tcache poisoning, double free, UAF, fastbin dup, and where to aim.
- **`pwn-methodology`** — _priority: high_
  - Path: [sa--instructions--binary-exploitation--pwn-methodology.md](https://raw.githubusercontent.com/BigBread57/zametki/main/12-pwn/sa--instructions--binary-exploitation--pwn-methodology.md)
  - Triggers: `["pwn", "binary exploitation", "checksec", "pwntools", "NX", "PIE", "canary", "RELRO", "libc leak", "ELF", "which bug class", "remote exploit", "GDB", "gef", "pwndbg"]`
  - Master decision tree for triaging a pwn binary from checksec to a working remote exploit.
- **`rop-chains`** — _priority: high_
  - Path: [sa--instructions--binary-exploitation--rop-chains.md](https://raw.githubusercontent.com/BigBread57/zametki/main/12-pwn/sa--instructions--binary-exploitation--rop-chains.md)
  - Triggers: `["ROP", "rop chain", "ROPgadget", "ropper", "ret2libc", "pop rdi", "puts@got", "GOT leak", "ret2csu", "ret2dlresolve", "one_gadget", "execve", "ORW", "gadget"]`
  - Build ROP chains — ret2libc via GOT leak, ret2csu/ret2dlresolve, one_gadget, and execve/ORW chains.
- **`stack-buffer-overflow`** — _priority: high_
  - Path: [sa--instructions--binary-exploitation--stack-buffer-overflow.md](https://raw.githubusercontent.com/BigBread57/zametki/main/12-pwn/sa--instructions--binary-exploitation--stack-buffer-overflow.md)
  - Triggers: `["buffer overflow", "stack overflow", "cyclic", "offset", "ret2win", "control RIP", "saved return address", "ret2shellcode", "gets", "movaps", "stack alignment", "smash the stack"]`
  - Find the overflow offset, take control of the saved return address, and pivot to ret2win or shellcode.

### crypto

- **`aes-block-cipher`** — _priority: high_
  - Path: [sa--instructions--crypto--aes-block-cipher.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--aes-block-cipher.md)
  - Triggers: `["AES", "AES ECB", "ECB", "CBC", "padding oracle", "bit flipping", "IV reuse", "CTR nonce reuse", "block cipher", "MODE_ECB", "duplicate blocks", "16-byte block"]`
  - Block-cipher attack playbook — ECB detection and byte-at-a-time decryption, CBC bit-flipping, padding oracle, IV reuse, and CTR/stream keystream reuse.
- **`crypto-triage`** — _priority: high_
  - Path: [sa--instructions--crypto--crypto-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--crypto-triage.md)
  - Triggers: `["unknown ciphertext", "base64", "hex", "decimal blob", "encoding", "what cipher", "gibberish string", "==", "0x", "identify crypto", "CyberChef", "magic"]`
  - First-look triage that sorts an unknown crypto challenge into encoding vs classical cipher vs modern crypto and routes to the right playbook.
- **`hash-cracking`** — _priority: high_
  - Path: [sa--instructions--crypto--hash-cracking.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--hash-cracking.md)
  - Triggers: `["hash", "hashcat", "john", "md5", "sha1", "sha256", "bcrypt", "ntlm", "crack password", "wordlist", "rockyou", "rules", "length extension", "hashpump", "identify hash"]`
  - Identify an unknown hash, pick the right hashcat/john mode, run wordlist+rule attacks, and perform hash length-extension forgery with hashpump.
- **`rsa-attacks`** — _priority: high_
  - Path: [sa--instructions--crypto--rsa-attacks.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--rsa-attacks.md)
  - Triggers: `["RSA", "n e c", "public key", "small e", "e=3", "cube root", "Wiener", "low d", "common modulus", "Hastad", "broadcast", "Fermat", "close primes", "factordb", "RsaCtfTool", "PEM"]`
  - Decision tree for RSA CTF challenges given n,e,c or a public key, from factordb lookup through low-exponent, Wiener, common-modulus, and Hastad attacks.
- **`xor-and-classical`** — _priority: high_
  - Path: [sa--instructions--crypto--xor-and-classical.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--xor-and-classical.md)
  - Triggers: `["XOR", "single-byte XOR", "repeating key", "Vigenere", "crib dragging", "known plaintext", "two-time pad", "Caesar", "ROT13", "substitution", "rail fence", "frequency analysis"]`
  - Recover plaintext from XOR (single-byte, repeating-key, keystream reuse) and classical ciphers (Caesar, Vigenere, substitution, rail fence) via frequency analysis and crib dragging.

### forensics-dfir

- **`disk-image-analysis`** — _priority: high_
  - Path: [sa--instructions--forensics-dfir--disk-image-analysis.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--disk-image-analysis.md)
  - Triggers: `["disk image", "E01", "dd image", "raw image", "autopsy", "sleuthkit", "MFT", "$MFT", "deleted file recovery", "registry hive", "browser history", "filesystem timeline", "NTFS", "shellbags", "prefetch", "amcache"]`
  - Mount and dissect Windows/Linux disk images - build a filesystem timeline, recover deleted files, and mine registry, browser, and execution artifacts for the investigative answer.
- **`forensics-triage`** — _priority: high_
  - Path: [sa--instructions--forensics-dfir--forensics-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--forensics-triage.md)
  - Triggers: `["forensics", "unknown file", "artifact identification", "file carving", "binwalk", "foremost", "exiftool", "steganography", "what kind of file is this", "triage", "hex dump", "magic bytes"]`
  - First-contact routing layer for any forensics artifact - identify the container type, extract embedded data, and dispatch to the correct specialist methodology (memory / disk / pcap / logs).
- **`log-analysis-sherlock`** — _priority: high_
  - Path: [sa--instructions--forensics-dfir--log-analysis-sherlock.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--log-analysis-sherlock.md)
  - Triggers: `["sherlock", "HackTheBox sherlock", "DFIR investigation", "incident timeline", "who what when how", "pivot on IOC", "correlate logs", "artifact bundle", "investigation questions", "blue team investigation"]`
  - End-to-end DFIR investigation workflow for HackTheBox Sherlocks and incident bundles - build one timeline across mixed artifacts, answer the who/what/when/where/how, and pivot on IOCs to reconstruct the full attack chain.
- **`memory-volatility`** — _priority: high_
  - Path: [sa--instructions--forensics-dfir--memory-volatility.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--memory-volatility.md)
  - Triggers: `["memory dump", "volatility", "volatility3", "RAM image", "vmem", "process injection", "malfind", "hashdump", "pslist", "netscan", "hiberfil", "MEMORY.DMP", "crash dump", "credential dump"]`
  - Volatility 3 driven RAM analysis - enumerate processes, hunt injected code and rogue network connections, dump processes/files, and recover credentials and flags from a Windows memory image.
- **`pcap-network-forensics`** — _priority: high_
  - Path: [sa--instructions--forensics-dfir--pcap-network-forensics.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--pcap-network-forensics.md)
  - Triggers: `["pcap", "pcapng", "wireshark", "tshark", "network capture", "follow stream", "extract files pcap", "DNS exfiltration", "C2 beaconing", "TLS keylog", "SSLKEYLOGFILE", "HTTP credentials", "FTP", "zeek", "network forensics"]`
  - Dissect a packet capture - filter to the relevant flows, follow and reassemble streams, carve transferred files and credentials, decrypt TLS, and detect exfiltration and C2 beaconing.
- **`soc-alert-triage`** — _priority: high_
  - Path: [sa--instructions--forensics-dfir--soc-alert-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--soc-alert-triage.md)
  - Triggers: `["SOC", "alert triage", "SIEM alert", "true positive", "false positive", "MITRE ATT&CK mapping", "IOC reputation", "VirusTotal", "AbuseIPDB", "escalation", "containment", "phishing alert", "EDR alert", "enrichment"]`
  - SOC analyst alert-handling workflow - enrich, decide true vs false positive, map to MITRE ATT&CK, reputation-check IOCs, and make the escalate/contain call.
- **`threat-hunting-detection`** — _priority: medium_
  - Path: [sa--instructions--forensics-dfir--threat-hunting-detection.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--threat-hunting-detection.md)
  - Triggers: `["threat hunting", "hypothesis-driven hunt", "sigma rule", "detection engineering", "ATT&CK hunt", "KQL", "Splunk SPL", "hunt query", "detection rule", "behavioral detection", "purple team"]`
  - Hypothesis-driven threat hunting and detection engineering - form ATT&CK-based hypotheses, hunt with Sigma/KQL/SPL query patterns, and convert findings into durable detections.
- **`windows-event-logs`** — _priority: high_
  - Path: [sa--instructions--forensics-dfir--windows-event-logs.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--windows-event-logs.md)
  - Triggers: `["evtx", "event log", "Security.evtx", "Event ID", "4624", "4625", "4688", "4720", "7045", "1102", "sysmon", "4104", "powershell log", "logon type", "lateral movement", "chainsaw", "hayabusa", "EvtxECmd"]`
  - Parse Windows EVTX logs and interpret the key Security, System, and Sysmon Event IDs to reconstruct logons, execution, persistence, and lateral movement.

### malware-analysis

- **`maldoc-scripts`** — _priority: high_
  - Path: [sa--instructions--malware-analysis--maldoc-scripts.md](https://raw.githubusercontent.com/BigBread57/zametki/main/13-malware/sa--instructions--malware-analysis--maldoc-scripts.md)
  - Triggers: `["macro", "maldoc", "malicious document", "docm", "xlsm", "olevba", "pdf malware", "pdfid", "peepdf", "lnk file", "hta", "obfuscated vbs", "obfuscated javascript", "phishing attachment", "excel 4 macro", "xlm"]`
  - Triage and deobfuscate document- and script-borne malware - Office macros, RTF, PDF, LNK, HTA, JS/VBS/PowerShell - without ever opening the file in its native application.
- **`malware-dynamic-sandbox`** — _priority: high_
  - Path: [sa--instructions--malware-analysis--malware-dynamic-sandbox.md](https://raw.githubusercontent.com/BigBread57/zametki/main/13-malware/sa--instructions--malware-analysis--malware-dynamic-sandbox.md)
  - Triggers: `["sandbox", "detonate", "dynamic analysis", "behavioral analysis", "procmon", "regshot", "fakenet", "inetsim", "c2 traffic", "network ioc", "persistence", "any.run", "cuckoo", "run the malware"]`
  - Controlled detonation of a sample in an isolated lab to capture process, file, registry, and network behavior without leaking traffic to the real internet.
- **`malware-static-triage`** — _priority: high_
  - Path: [sa--instructions--malware-analysis--malware-static-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/13-malware/sa--instructions--malware-analysis--malware-static-triage.md)
  - Triggers: `["suspicious exe", "PE analysis", "unknown binary", "static analysis", "strings", "imports", "entropy", "file hash", "virustotal", "capa", "pestudio", "reverse engineering triage", "dll analysis", "shellcode"]`
  - Safe, no-execution triage of an unknown binary - identify, hash, fingerprint, and enumerate capabilities before ever considering detonation.
- **`unpacking-deobfuscation`** — _priority: high_
  - Path: [sa--instructions--malware-analysis--unpacking-deobfuscation.md](https://raw.githubusercontent.com/BigBread57/zametki/main/13-malware/sa--instructions--malware-analysis--unpacking-deobfuscation.md)
  - Triggers: `["packed", "packer", "UPX", "high entropy", "obfuscated powershell", "base64 payload", "gzip deflate layer", "OEP", "dump process memory", "crypter", "de4dot", "confuserex", "cyberchef", "deobfuscate", "unpack"]`
  - Detect the packing/obfuscation layer, strip it (automated, dynamic dump, or manual OEP), and recover clean code or plaintext script for real analysis.
- **`yara-iocs`** — _priority: high_
  - Path: [sa--instructions--malware-analysis--yara-iocs.md](https://raw.githubusercontent.com/BigBread57/zametki/main/13-malware/sa--instructions--malware-analysis--yara-iocs.md)
  - Triggers: `["yara", "write a yara rule", "signature", "ioc", "indicators of compromise", "detection rule", "sigma", "capa", "hunting", "threat intel", "family attribution", "imphash"]`
  - Turn analyzed samples into durable detection - extract high-quality IOCs, write and test YARA rules, map capabilities with capa, and express behavior as Sigma.

### hardware

- **`firmware-analysis`** — _priority: high_
  - Path: [sa--instructions--hardware--firmware-analysis.md](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--firmware-analysis.md)
  - Triggers: `["firmware", ".bin", "firmware image", "binwalk", "squashfs", "jffs2", "cramfs", "ubi", "router firmware", "extract filesystem", "hardcoded password", "backdoor", "U-Boot image", "entropy", "emulate firmware"]`
  - Extract, unpack, and analyze a firmware image — carve filesystems, hunt creds/keys/backdoors, emulate, and pull the flag.
- **`hardware-triage`** — _priority: high_
  - Path: [sa--instructions--hardware--hardware-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--hardware-triage.md)
  - Triggers: `["PCB photo", "board photo", "chip markings", "debug header", "test pads", "unknown board", "identify interface", "hardware challenge", "which pins", "logic analyzer", "multimeter"]`
  - First-look triage of a PCB photo or physical board — ID chips, find debug/interface pins, and route to the correct sub-playbook.
- **`rf-sdr`** — _priority: medium_
  - Path: [sa--instructions--hardware--rf-sdr.md](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--rf-sdr.md)
  - Triggers: `["SDR", "RF", "433MHz", "315MHz", "keyfob", "garage door", "remote", "rtl_sdr", "hackrf", "gqrx", "GNU Radio", "URH", "rtl_433", "ASK", "OOK", "FSK", "IQ capture", "replay attack", ".cu8", ".wav signal"]`
  - Capture, identify, demodulate, and replay ISM-band RF (433/315/868/915 MHz) — decode keyfobs/remotes and pull flags from signals.
- **`side-channel-glitch`** — _priority: medium_
  - Path: [sa--instructions--hardware--side-channel-glitch.md](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--side-channel-glitch.md)
  - Triggers: `["side channel", "power analysis", "DPA", "CPA", "SPA", "fault injection", "glitch", "voltage glitch", "clock glitch", "ChipWhisperer", "recover AES key", "bypass PIN", "secure boot bypass", "timing attack", "power trace"]`
  - Recover keys via power analysis (SPA/DPA/CPA) and bypass checks via voltage/clock fault injection with ChipWhisperer.
- **`spi-i2c-flash-dump`** — _priority: high_
  - Path: [sa--instructions--hardware--spi-i2c-flash-dump.md](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--spi-i2c-flash-dump.md)
  - Triggers: `["SPI flash", "25Q64", "W25Q128", "SOIC-8", "SOIC clip", "24C02", "AT24", "I2C EEPROM", "flashrom", "CH341A", "Bus Pirate", "FT2232", "chip-off", "dump flash", "read eeprom"]`
  - Identify and dump SPI NOR / I2C EEPROM flash chips (in-circuit or chip-off) with flashrom, then carve the firmware.
- **`uart-jtag-swd`** — _priority: high_
  - Path: [sa--instructions--hardware--uart-jtag-swd.md](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--uart-jtag-swd.md)
  - Triggers: `["UART", "serial console", "TX RX", "USB-TTL", "screen", "picocom", "baud rate", "U-Boot prompt", "JTAG", "SWD", "OpenOCD", "JTAGulator", "halt CPU", "dump memory", "root shell"]`
  - Get a serial console or on-chip debug — find UART/JTAG/SWD pins, drop to a root shell or U-Boot, halt the CPU and dump memory/flash.

### stego-osint-misc

- **`misc-jail-escapes`** — _priority: high_
  - Path: [sa--instructions--stego-osint-misc--misc-jail-escapes.md](https://raw.githubusercontent.com/BigBread57/zametki/main/17-misc/sa--instructions--stego-osint-misc--misc-jail-escapes.md)
  - Triggers: `["python jail", "pyjail", "sandbox escape", "bash restricted", "rbash", "restricted shell", "filter bypass", "eval", "exec", "blacklist bypass", "command injection", "escape the jail", "builtins", "subclasses", "misc recon"]`
  - Decision-tree methodology for escaping Python jails, restricted shells (rbash), and filtered command sandboxes in misc CTF challenges.
- **`osint`** — _priority: high_
  - Path: [sa--instructions--stego-osint-misc--osint.md](https://raw.githubusercontent.com/BigBread57/zametki/main/15-osint/sa--instructions--stego-osint-misc--osint.md)
  - Triggers: `["osint", "find the person", "find the place", "find the location", "geolocation", "who is", "username", "email", "social media", "reverse image search", "exif gps", "google dorking", "wayback machine", "identify the building", "where was this photo taken"]`
  - Decision-tree methodology for OSINT CTF challenges - finding people, usernames, emails, and geolocating photos from open sources.
- **`steganography`** — _priority: high_
  - Path: [sa--instructions--stego-osint-misc--steganography.md](https://raw.githubusercontent.com/BigBread57/zametki/main/10-steganography/sa--instructions--stego-osint-misc--steganography.md)
  - Triggers: `["image", "png", "jpg", "jpeg", "bmp", "wav", "audio", "spectrogram", "steghide", "zsteg", "stegsolve", "outguess", "binwalk", "exiftool", "hidden data", "least significant bit", "lsb", "hidden message", "aperisolve", "appended data"]`
  - Decision-tree methodology for extracting hidden data from images, audio, and documents in CTF steganography challenges.

---

## Global operating loop (from `_meta/00-global-methodology.md`)
1. **Classify** the challenge type → pick a category.
2. **Enumerate broadly, then narrow** — map the full attack surface before exploiting.
3. **One falsifiable hypothesis at a time** — test the cheapest disproof first; kill dead leads fast (rabbit-hole guard).
4. **Log everything** — every credential, hash, path, version, endpoint; these connect stages.
5. **Escalate the loop** — foothold → local enum → privesc → loot → pivot; re-enumerate after each privilege change.
6. **Capture with proof** — display the flag; never report success on inference.

## Conventions
- **Methodology only** — ordered, falsifiable steps; no theory dumps.
- **Copy-paste commands** with `<angle-bracket>` placeholders.
- **Cross-links** use `file-name (ссылка источника; файл не найден в библиотеке)` matching another file's `name:` slug.
- Each file has a **Common pitfalls** section — obey the time-box and avoid the listed rabbit holes.

## Install / use

This folder **is** the skill — [sa--SKILL.md](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/sa--SKILL.md), `instructions/`, and the trigger indexes all live together.

**Claude / Claude Code:** copy this whole folder into `~/.claude/skills/` (global) or `<project>/.claude/skills/` (project), renaming the folder to `ctf-methodology` (must match the skill `name`). Claude auto-discovers it; then just say e.g. *"use the ctf-methodology skill on this target."*

**XHack AI Agent ([xhack.io](https://xhack.io)):** open the agent → **Plugins / Skills** → **Upload** the zipped folder (or **Select folder**). XHack loads `triggers.min.json` and routes automatically.

**Any agent / manual:** parse `triggers.min.json` (~5k tokens = the whole routing table), match triggers, then load the one `instructions/...` playbook it points to. Helper routers are in `router/` (`route.py`, `route.js`).

All paths in the index above are relative to this [sa--SKILL.md](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/sa--SKILL.md), so the folder is fully self-contained.


---

## Зависимости и ссылки вне библиотеки

Следующие файлы или ресурсы упомянуты в исходнике, но не входят в эту библиотеку документов. Их наличие и работоспособность здесь не проверялись.

- `router/route.js`
- `router/route.py`
- `triggers.min.json`
