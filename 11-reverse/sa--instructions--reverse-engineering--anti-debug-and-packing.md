# Anti-Debug & Packing

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/reverse-engineering/anti-debug-and-packing.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/reverse-engineering/anti-debug-and-packing.md)

---

---
name: anti-debug-and-packing
category: reverse-engineering
triggers: ["packed", "UPX", "high entropy", "anti-debug", "ptrace", "PTRACE_TRACEME", "detect debugger", "timing check", "obfuscation", "unpack", "manual unpack", "LD_PRELOAD", "segfault under gdb"]
tools: [upx, gdb, pwndbg, "LD_PRELOAD", ltrace, strace, binwalk, "die (Detect It Easy)", radare2, "readelf"]
priority: high
description: Detect and defeat packers (UPX/custom) and anti-debug tricks (ptrace, timing, /proc checks) to reach analyzable code.
---
# Anti-Debug & Packing

## Trigger context
Triage flagged high entropy, few imports, `UPX0/UPX1` sections, or the binary behaves differently under a debugger (exits, segfaults, prints "debugger detected", or silently takes a wrong path). You must unwrap/neutralize before real RE.

## Objective
- Recognize packing/obfuscation and get an unpacked, disassemblable image.
- Identify and bypass anti-debug so gdb/ltrace can observe the real logic.
- Restore a workflow where re-triage-static / ghidra-workflow / re-dynamic-gdb apply.

## Methodology (decision tree)
1. Detect packing: `die`/`rabin2 -I` report a packer; `readelf -S` shows `UPX0/UPX1`; `upx -t` says OK; entropy is high and imports are near-empty.
2. UPX specifically -> try `upx -d` (trivial). If it fails (headers/magic stripped to break upx -d), do a manual unpack (dump from memory at the OEP).
3. Custom/unknown packer -> generic "run until unpacked, then dump" approach: break at the tail jump to the original entry point (OEP), dump the process memory, rebuild.
4. After unpacking, re-run re-triage-static on the dumped file.
5. Anti-debug (binary misbehaves under gdb / ptrace visible in strace) -> pick a bypass: LD_PRELOAD stub, gdb `catch syscall ptrace` + force return 0, or patch the check.
6. Timing / `/proc/self/status` (TracerPid) / parent-name checks -> patch or preload; see Branching logic.

## Commands & payloads
```bash
# --- detect ---
die ./chall               # Detect It Easy: names the packer/compiler
rabin2 -I ./chall | grep -i -E 'packer|lang|static'
readelf -S ./chall | grep -iE 'UPX|packed'
upx -t ./chall            # "[OK]" => UPX
python3 -c "import math,sys,collections as c;d=open('chall','rb').read();\
p=[v/len(d) for v in c.Counter(d).values()];\
print('entropy',-sum(x*math.log2(x) for x in p))"   # ~7.9+ => packed/encrypted

# --- UPX automatic ---
upx -d ./chall -o chall.unpacked          # done if magic intact
upx -d -f ./chall -o chall.unpacked       # force

# --- UPX manual unpack (when upx -d refuses) ---
# UPX stub decompresses in memory then JMPs to the OEP. Break at that tail jump, dump.
gdb ./chall
#  (pwndbg) entry            # stop at packed entry / or: starti
#  (pwndbg) # single-step/scan for the final 'jmp' out of the UPX0 stub region,
#           # or set an hbreak on the OEP once identified in the Listing
#  (pwndbg) # once at OEP, dump the now-unpacked image:
#  (pwndbg) vmmap           # find the module range
#  (pwndbg) dump memory chall.dump 0x400000 0x452000   # adjust to mapped range
# then reconstruct sections/imports with an unpacker helper if needed (e.g. repack notes)
```
```c
/* --- ptrace anti-debug bypass via LD_PRELOAD --- */
/* Many Linux binaries call ptrace(PTRACE_TRACEME,0,0,0); a nonzero/-1 result
   means "already traced" => debugger present. Stub ptrace to always return 0. */
/* noptrace.c */
#include <stdio.h>
long ptrace(long request, long pid, long addr, long data) {
    (void)request;(void)pid;(void)addr;(void)data;
    return 0;   /* pretend: not being traced, call succeeded */
}
```
```bash
gcc -shared -fPIC noptrace.c -o noptrace.so
LD_PRELOAD=./noptrace.so ./chall          # run with ptrace neutralized
# inside gdb:
gdb ./chall
#  (gdb) set environment LD_PRELOAD ./noptrace.so
#  (gdb) run

# --- alternative: intercept ptrace in gdb without a preload ---
gdb ./chall
#  (gdb) catch syscall ptrace
#  (gdb) run
#  (gdb) # on syscall-return stop, force success:
#  (gdb) set $rax = 0
#  (gdb) continue
#  (gdb) # or make it permanent:
#  (gdb) commands            # auto-run on each catch
#  >  set $rax = 0
#  >  continue
#  >  end

# --- defeat /proc/self/status TracerPid check (grep for the open) ---
strace -f ./chall 2>&1 | grep -E 'ptrace|/proc/self/status|/proc/self/stat'
# then patch the comparison in gdb (set the compared value to 0) or nop the branch.

# --- timing checks (rdtsc / clock/nanosleep gate) ---
# Under a debugger, elapsed time balloons and the check fails. Bypass by patching the
# time-delta comparison branch, or by NOPping the rdtsc-diff test in gdb:
#  (gdb) set {char}0xADDR = 0x90   # nop out the anti-debug jump, or flip je/jne
```

## Branching logic
- `upx -t` OK -> `upx -d`; if refused, manual dump at OEP, then re-triage.
- Not UPX but high entropy + tiny import table -> custom packer: generic OEP-dump; look for the big memory allocation + write loop then a tail jump.
- Binary exits immediately under gdb but runs bare -> almost certainly `ptrace(PTRACE_TRACEME)`: LD_PRELOAD stub or gdb ptrace catch.
- Binary checks `/proc/self/status` TracerPid -> preload can't fix a file read; patch the comparison branch instead.
- Anti-debug via signals (installs SIGTRAP handler so int3 breakpoints are swallowed) -> use hardware breakpoints (`hbreak`) and `handle SIGTRAP nostop pass`.
- Parent-process/`getppid` name check ("gdb"/"ltrace") -> rename the debugger or patch the strcmp.
- Self-modifying/decrypting code -> set breakpoints AFTER decryption (on the region once it's written), not on the still-encrypted bytes.

## Common pitfalls
- LD_PRELOAD only overrides dynamically-resolved libc symbols; it does NOT help against a direct `syscall` to ptrace (syscall number 101) or a statically-linked binary — patch or use the gdb catch instead.
- Dumping at the wrong moment (before decompression finishes) yields still-packed garbage; confirm you're at the OEP (real code/strings visible).
- A manual dump often has a broken import table; you may need to fix it before Ghidra shows clean calls, or just analyze the dumped code region directly.
- gdb disables ASLR by default, changing addresses vs a bare run — recompute OEP/offsets from `vmmap`.
- Multiple stacked anti-debug checks: bypassing one isn't enough; strace/trace for every ptrace/proc/time gate.
- `int3` software breakpoints can be detected (0xCC scanning) or eaten by a SIGTRAP handler; prefer hardware breakpoints on hardened targets.

## Proof / verification
- After `upx -d`/dump: `file`, `strings`, and `readelf -S` on the result should show normal sections, readable strings, and a populated import table.
- After anti-debug bypass: the binary now reaches and executes the real check under gdb (you can break on `strcmp`/the validator and see live operands).
- Confirm entropy dropped post-unpack (recompute; expect well below ~7.5 for normal code+data).

## Cross-references
- After unpacking, restart fingerprinting -> [sa--instructions--reverse-engineering--re-triage-static.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--re-triage-static.md)
- Read the unpacked logic -> [sa--instructions--reverse-engineering--ghidra-workflow.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--ghidra-workflow.md)
- Live breakpointing/patching once debuggable -> [sa--instructions--reverse-engineering--re-dynamic-gdb.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--re-dynamic-gdb.md)
- Heavy math check better solved offline -> [sa--instructions--reverse-engineering--angr-symbolic-execution.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--angr-symbolic-execution.md)

## Credible sources
- CTF-wiki: detect & bypass debugging (Linux): https://ctf-wiki.mahaloz.re/reverse/linux/detect-dbg/
- HackTricks anti-debugging: https://book.hacktricks.wiki/en/reversing/common-api-used-in-malware.html
- UPX project: https://upx.github.io/
- Bypassing ptrace with LD_PRELOAD (analysis-of-anti-analysis): https://github.com/yellowbyte/analysis-of-anti-analysis
- Linux anti-debugging (Auberger): https://seblau.github.io/posts/linux-anti-debugging
