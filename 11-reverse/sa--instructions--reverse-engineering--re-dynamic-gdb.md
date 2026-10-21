# RE Dynamic Analysis with GDB

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/reverse-engineering/re-dynamic-gdb.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/reverse-engineering/re-dynamic-gdb.md)

---

---
name: re-dynamic-gdb
category: reverse-engineering
triggers: ["gdb", "pwndbg", "GEF", "breakpoint", "dynamic analysis", "runtime", "patch comparison", "bypass check", "ltrace", "strace", "examine memory", "license check", "password check", "step through"]
tools: [gdb, pwndbg, GEF, ltrace, strace, "gdb set", "gdb break", objdump, radare2]
priority: high
description: Drive a binary at runtime under gdb/pwndbg/GEF to watch comparisons, read secrets from memory, and patch or bypass checks live.
---
# RE Dynamic Analysis with GDB

## Trigger context
Static reading stalls: the flag/password is derived at runtime, obfuscated, or gated behind a check you want to force. Use dynamic analysis to observe real register/memory values at the decisive comparison and to bypass gates without fully understanding them.

## Objective
- Break at the comparison/decision point and read the operands (the expected value is usually sitting in a register or buffer).
- Dump runtime-decrypted strings/keys from memory.
- Flip a branch, patch a return value, or force the "success" path to reveal the flag.

## Methodology (decision tree)
1. Reconnoiter without a debugger first: `ltrace ./bin` often prints the `strcmp(input, "SECRET")` args directly. `strace` shows syscalls (file reads, `ptrace`, timing).
2. Load in gdb+pwndbg/GEF. Find the check: `break main`, then look for `strcmp/strncmp/memcmp` or `cmp` against a constant near the input-handling code.
3. Break at the comparison. Inspect operands: for `strcmp`, args are in RDI/RSI (x86-64 SysV). `x/s $rsi` frequently is the expected string.
4. If the secret is computed, break AFTER the transform and dump the buffer.
5. To bypass: (a) `set $eax=0` / set ZF to force the taken branch, or (b) patch the compare/jump, or (c) `return` early from a validator with a chosen value.
6. Anti-debug present (process exits/segfaults only under gdb)? -> anti-debug-and-packing.

## Commands & payloads
```bash
# recon before gdb
ltrace ./chall            # library calls + args: reveals strcmp/strncmp/memcmp operands
ltrace -s 128 ./chall     # longer string args
strace -f ./chall         # syscalls: open/read of key files, ptrace, nanosleep timing

gdb ./chall               # pwndbg/GEF auto-load if installed (~/.gdbinit)
```
```bash
# --- inside gdb (pwndbg / GEF) ---
info functions            # list symbols (if not stripped)
break main                # or: break *0x401234   (absolute)  |  break *main+42
run  arg1                 # run [args]; 'run < input.txt' to feed stdin
starti                    # stop at first instruction (before libc init)

# breakpoint on a libc call to catch the comparison
break strcmp
break strncmp
break memcmp
continue

# at the strcmp break, read operands (x86-64 SysV: RDI, RSI, RDX)
x/s $rdi                  # arg1 (your input)
x/s $rsi                  # arg2 (often the EXPECTED secret/flag)
info args                 # if symbols/debuginfo present

# examine memory / registers
info registers            # or pwndbg: 'regs'
x/16xg $rsp               # 16 giant words from stack
x/8i $pc                  # next 8 instructions
telescope $rsp 20         # pwndbg: dereference-chain view
vmmap                     # pwndbg/GEF: memory map (find base for PIE offsets)
search --string flag{     # pwndbg: scan mapped memory for a string
grep flag                 # GEF: 'grep flag' searches memory

# step / continue
ni / si                   # step over / step into one instruction
finish                    # run until current function returns (see retval in RAX)
stepi 5                   # N steps

# --- BYPASS a check ---
# 1) force a return value from a validator, then jump past it
break *validate+120       # just before the compare result is used
set $rax = 1              # or 0 — whichever means "valid"
# 2) flip the CPU flags to take/skip a conditional jump
p $eflags                 # inspect; ZF is bit 6 (0x40)
set $eflags |= 0x40       # set Zero Flag  -> makes je taken / jne skipped
set $eflags &= ~0x40      # clear Zero Flag
# 3) patch a byte at runtime (turn jne into je, or nop a call)
set {char}0x401337 = 0x74 # write opcode byte (0x74 = JE short)
# 4) skip an instruction/call by moving PC
set $pc = 0x401360

# handle PIE: compute real address = base + static_offset
vmmap                     # note the module base (e.g. 0x555555554000)
break *0x555555554000+0x1234
```

## Branching logic
- `strcmp`/`memcmp` comparison found -> read RSI (expected) -> submit that; done.
- Secret is decrypted in a loop -> break after the loop, `x/s` the output buffer.
- Multiple checks chained -> patch each branch to success, or set the final "win" flag var and jump to the print-flag block.
- Length-checked input -> break on the length compare too; the required length constrains angr/bruteforce fallback.
- Binary dies/exits under gdb but runs standalone -> ptrace anti-debug: `catch syscall ptrace`, then `set $rax=0` on return, or LD_PRELOAD stub (anti-debug-and-packing).
- Register calling convention differs (32-bit x86 uses the stack: args at `$esp+4`, `$esp+8`; ARM uses R0-R3) — adjust operand reads accordingly.

## Common pitfalls
- Wrong calling convention: x86-64 SysV = RDI,RSI,RDX,RCX,R8,R9; Windows x64 = RCX,RDX,R8,R9; x86 = stack. Reading the wrong register wastes time.
- With PIE/ASLR, absolute breakpoints from static analysis are wrong until you add the runtime base (`vmmap`). gdb disables ASLR by default (`set disable-randomization on`), so base is stable across `run`s locally but differs from remote.
- Breaking on `strcmp` catches EVERY strcmp, including libc-internal ones; add a condition or break at the specific call site.
- `set $eax=` only works while stopped; forcing a value that the code later recomputes is futile — patch at the last use.
- ltrace/strace are blocked or misled by static linking and anti-debug; a clean trace is not proof of anything.
- `finish` won't work if the frame is non-standard (tail calls, no frame pointer).

## Proof / verification
- After forcing success, confirm the program actually prints the real flag (not a hardcoded "Correct!" with no flag).
- Cross-check a memory-dumped secret by feeding it to the unmodified binary via `run < input`.
- Note the exact instruction address of the decisive branch so the bypass is reproducible.

## Cross-references
- Program only misbehaves under a debugger -> [sa--instructions--reverse-engineering--anti-debug-and-packing.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--anti-debug-and-packing.md)
- Need the decompiled logic to know WHERE to break -> [sa--instructions--reverse-engineering--ghidra-workflow.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--ghidra-workflow.md)
- Deterministic input check better solved offline -> [sa--instructions--reverse-engineering--angr-symbolic-execution.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--angr-symbolic-execution.md)
- Initial fingerprint/checksec/PIE status -> [sa--instructions--reverse-engineering--re-triage-static.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--re-triage-static.md)

## Credible sources
- pwndbg docs & commands: https://pwndbg.re/pwndbg/latest/commands/
- pwndbg cheatsheet (PDF): https://pwndbg.re/CHEATSHEET.pdf
- GEF documentation: https://hugsy.github.io/gef/
- GDB manual (breakpoints, altering execution): https://sourceware.org/gdb/current/onlinedocs/gdb/
- CTF101 debugging with GDB: https://ctf101.org/reverse-engineering/what-is-gdb/
