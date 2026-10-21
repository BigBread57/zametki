# Networking — tools reference

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / knowledge/tools-reference/network.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/knowledge/tools-reference/network.md)

---

# Networking — tools reference

## Live connection challenges (ssh / netcat)
```
nc <host> <port>                # connect and read first — many CTF services hand over the flag immediately, no input needed
ssh <user>@<host> -p <port>       # check the login banner and home directory before searching further
```
Extract host/port/credentials directly from the challenge description
(`ssh as USER to HOST at port PORT`, `nc HOST PORT`, `USER@HOST -p PORT` are the
common phrasings) rather than assuming — get this right before connecting.

## Scanning (only against the explicit challenge target — see CLAUDE.md hard rules)
```
nmap -sV -p- <host>              # full port sweep with version detection
nmap -sC -sV -p <ports> <host>    # default scripts + version, once ports are known
```

## Packet capture analysis
```
tshark -r <cap>.pcap -q -z conv,tcp     # conversation summary — fastest way to see what's in a capture
tshark -r <cap>.pcap -Y "tcp.stream eq 0" -T fields -e data   # dump one stream's raw hex
tshark -r <cap>.pcap --export-objects http,out/                 # pull files transferred over HTTP
```
See [na--knowledge--tools-reference--forensics.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/na--knowledge--tools-reference--forensics.md) for the fuller pcap workflow —
these overlap heavily; use whichever file you're already in.

## Raw sockets from Python (once a plain nc script isn't enough)
```
.venv/bin/python -c "
from pwn import remote
r = remote('<host>', <port>)
print(r.recvline())
r.sendline(b'...')
"
```
`pwntools`' `remote()` is usually less friction than hand-rolling `socket` calls
for anything beyond a one-shot read.
