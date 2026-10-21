# SSH Tunneling and Port Forwarding

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/access-pivoting/tunneling-port-forwarding.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/access-pivoting/tunneling-port-forwarding.md)

---

---
name: tunneling-port-forwarding
category: access-pivoting
triggers: ["port forward", "ssh -L", "ssh -R", "ssh -D", "SOCKS proxy", "tunnel", "internal service", "localhost only", "bind 127.0.0.1", "reverse tunnel", "sshuttle"]
tools: ["ssh", "sshuttle", "socat", "netstat", "ss", "curl", "nmap"]
priority: high
description: Reach services bound to an internal interface by building SSH local, remote, and dynamic (SOCKS) tunnels from a foothold.
---
# SSH Tunneling and Port Forwarding

## Trigger context
Use when a service is bound to `127.0.0.1` or an internal subnet you cannot reach directly: a DB on `localhost:3306`, an admin panel on `10.10.20.5:8080`, or an nmap that shows a port "filtered" from your box but "open" from the foothold. Also when you need to expose YOUR attacker service to a target that cannot route back to you (reverse tunnel through NAT).

## Objective
Move packets between your attack box and an internal service through an SSH channel, choosing the minimal forward: a single port (`-L`), a callback port (`-R`), or a whole-network SOCKS proxy (`-D`). Verify the tunnel carries real traffic before layering tools on top.

## Methodology (decision tree)
1. **Need one internal port on your box?** Local forward `-L`. Simplest; one service at a time.
2. **Need the target to reach a port on your box (target is behind NAT / you only have a reverse shell out)?** Remote forward `-R`.
3. **Need to reach many internal hosts/ports (scan a subnet, browse many apps)?** Dynamic forward `-D` = a SOCKS proxy; pair with proxychains/browser.
4. **Want the whole subnet routed transparently without SOCKS?** `sshuttle` (a "poor man's VPN" over SSH).
5. **No SSH server on the foothold?** Fall back to chisel/ligolo — see pivoting file.
Always confirm which interface the service binds (`ss -tlnp`) so you forward the right address.

## Commands & payloads
```bash
# --- Local forward (-L): pull an internal service to YOUR localhost ---
# Syntax: -L [bind:]localport:targethost:targetport
ssh -L 8080:127.0.0.1:8080 user@foothold          # foothold's own localhost:8080 -> your :8080
ssh -L 3306:10.10.20.5:3306 user@foothold         # internal DB host -> your :3306
curl http://127.0.0.1:8080                         # now hits the internal app
# -N = no shell (just the tunnel), -f = background, -g = allow others to use your local port
ssh -fN -L 8080:127.0.0.1:8080 user@foothold

# --- Remote forward (-R): expose YOUR box to the target network ---
# Syntax: -R [bind:]remoteport:localhost:localport   (runs on the SSH server)
ssh -R 4444:127.0.0.1:4444 user@foothold           # target's :4444 -> your listener :4444
# Use case: your metasploit/nc listener is on the attack box, target can only reach the foothold.
# Needs "GatewayPorts yes" on the server to bind 0.0.0.0 instead of just its loopback.

# --- Dynamic forward (-D): a SOCKS proxy over SSH ---
ssh -D 1080 -fN user@foothold                      # SOCKS5 proxy on your 127.0.0.1:1080
# Then point tools at it:
curl --socks5-hostname 127.0.0.1:1080 http://10.10.20.5/
proxychains nmap -sT -Pn 10.10.20.0/24             # via proxychains (socks4/5 1080)
# Browser: set SOCKS5 host 127.0.0.1 port 1080, proxy DNS through it.

# --- sshuttle: transparent subnet VPN over SSH (no SOCKS needed) ---
sshuttle -r user@foothold 10.10.20.0/24            # route the whole subnet through SSH
sshuttle -r user@foothold 0.0.0.0/0 --dns          # everything, incl. DNS (loud)

# --- Add a forward to an existing session (SSH escape sequence) ---
# Press:  Enter  ~C   then at the ssh> prompt:
#   -L 9090:127.0.0.1:9090

# --- Verify what is listening / bound where ---
ss -tlnp                                            # on the foothold: which svc, which interface
netstat -ano                                        # (Windows foothold equivalent)
```

## Branching logic
- Forward opens but `curl` hangs -> the foothold cannot reach `targethost:targetport`; test from the foothold shell (`curl`/`nc -vz`) first.
- `-R` binds only to server loopback -> server needs `GatewayPorts yes` (or `clientspecified`); otherwise the callback only works from the foothold itself.
- `bind: Address already in use` -> local port taken; pick another (`-L 8081:...`).
- SOCKS + nmap gives no results -> use `-sT` (full connect) and `-Pn`; SOCKS cannot carry raw/SYN scans or ICMP.
- DNS resolves wrong host -> use `--socks5-hostname` (curl) or `proxy_dns` (proxychains) so names resolve on the far side.
- Need many ports/hosts -> stop stacking `-L`; switch to `-D` SOCKS or sshuttle.

## Common pitfalls
- Confusing `-L` and `-R` direction: `-L` = pull toward you, `-R` = push a listener onto the target side.
- Forgetting the tunnel is torn down when the SSH session dies; use `-fN` (and `autossh` / `ServerAliveInterval` for stability).
- Forwarding `localhost` vs the real internal IP — `127.0.0.1` on the foothold is NOT the internal host; specify the actual `10.x` address.
- SOCKS proxies do only TCP connect scans; expecting UDP/SYN/ICMP through them fails silently.
- Leaving `-g`/`GatewayPorts` open exposes your tunnel to the whole network — scope-risky.

## Proof / verification
- `curl`/service banner returned through the forwarded local port matches the internal service.
- `ss -tlnp` on your box shows `ssh` listening on the forwarded port.
- For SOCKS: `proxychains curl http://<internal>` returns content the direct route cannot.
- For `-R`: the target successfully connects back to your listener (catch it with `nc -lvnp`).

## Cross-references
- [sa--instructions--access-pivoting--ssh-access-and-keys.md](https://raw.githubusercontent.com/BigBread57/zametki/main/07-access-pivoting/sa--instructions--access-pivoting--ssh-access-and-keys.md) — establishing the SSH session these tunnels ride on.
- [sa--instructions--access-pivoting--pivoting-proxychains.md](https://raw.githubusercontent.com/BigBread57/zametki/main/07-access-pivoting/sa--instructions--access-pivoting--pivoting-proxychains.md) — proxychains config, and chisel/ligolo for when no SSH server is available.

## Credible sources
- HackTricks — Tunneling and Port Forwarding: https://book.hacktricks.wiki/en/generic-methodologies-and-resources/tunneling-and-port-forwarding.html
- man ssh (-L / -R / -D): https://man.openbsd.org/ssh
- sshuttle documentation: https://sshuttle.readthedocs.io/en/stable/
- OpenSSH port forwarding guide (SSH.com): https://www.ssh.com/academy/ssh/tunneling/example
