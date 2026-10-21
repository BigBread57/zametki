# angr Symbolic Execution

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/reverse-engineering/angr-symbolic-execution.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/reverse-engineering/angr-symbolic-execution.md)

---

---
name: angr-symbolic-execution
category: reverse-engineering
triggers: ["angr", "symbolic execution", "constraint solving", "flag checker", "explore find avoid", "z3", "SMT", "brute force alternative", "license check", "keygen", "deterministic checker"]
tools: [angr, claripy, z3, python3, "simulation manager", SimProcedure]
priority: high
description: Use angr symbolic execution to auto-solve deterministic input->accept flag checkers via explore(find/avoid), hooking, and constraint solving.
---
# angr Symbolic Execution

## Trigger context
The binary is a self-contained checker: it reads input, runs a deterministic transform/comparison, and prints "Correct"/"Wrong" (or returns 0/1). Rather than reverse the math by hand, let angr explore paths and have the SMT solver produce the input that reaches "win". Best when the state space is bounded (fixed-length input, no heavy crypto, no network/randomness).

## Objective
- Reach the success address (or success output) and read back the input that got there.
- Constrain symbolic input (charset, length, printable) so the solver returns a usable flag.
- Hook away expensive/irrelevant functions to keep exploration tractable.

## Methodology (decision tree)
1. Decide angr fits: deterministic, offline check; input length known/bounded; no strong crypto (SHA/AES) — those explode the solver. If crypto, reverse it manually or z3 the math directly.
2. Get target addresses from Ghidra/objdump: the "print success" block (find) and "print failure"/exit block(s) (avoid).
3. Choose input model:
   - stdin -> `entry_state(stdin=...)` or symbolic file, read result from `posix.dumps(0)`.
   - argv -> `entry_state(args=[path, symbolic_bytes])`.
   - direct buffer -> `blank_state` at the check function + `state.memory.store`.
4. Add constraints on the symbolic bytes (printable/charset, length, null terminator).
5. `simgr.explore(find=..., avoid=...)`; if it stalls, prune with avoid, hook slow funcs, or use `find=lambda s: b"Correct" in s.posix.dumps(1)`.
6. On success, `simgr.found[0].posix.dumps(0)` (or eval the symbolic var) = the flag.
7. Blow-up (state explosion / memory) -> narrow with more avoids, LAZY_SOLVES, veritesting, or hook the offending routine.

## Commands & payloads
```bash
pip install angr            # brings in claripy + z3
python3 solve.py
```
```python
# solve.py -- canonical stdin flag-checker solver
import angr, claripy

PROJ_BASE = 0x400000                 # for non-PIE; for PIE use base 0 + load opts
proj = angr.Project('./chall', auto_load_libs=False)   # skip libc for speed

FLAG_LEN = 32
flag = claripy.BVS('flag', FLAG_LEN * 8)               # symbolic bitvector
# feed it as stdin (a symbolic file backing fd 0)
state = proj.factory.entry_state(
    stdin=flag,
    add_options={angr.options.SYMBOL_FILL_UNCONSTRAINED_MEMORY,
                 angr.options.SYMBOL_FILL_UNCONSTRAINED_REGISTERS})

# constrain each byte to printable ASCII (helps the solver + avoids junk)
for i in range(FLAG_LEN):
    b = flag.get_byte(i)
    state.solver.add(b >= 0x20, b <= 0x7e)

simgr = proj.factory.simgr(state)

FIND  = 0x401337      # address of the "Correct!"/win block (from Ghidra)
AVOID = 0x401300      # address of the "Wrong"/fail block(s)
simgr.explore(find=FIND, avoid=AVOID)

if simgr.found:
    sol = simgr.found[0]
    print("FLAG:", sol.posix.dumps(0))       # bytes fed to stdin
    # or: print(sol.solver.eval(flag, cast_to=bytes))
else:
    print("no solution — loosen constraints / check addrs")
```
```python
# Output-based find/avoid (no exact address needed)
simgr.explore(find=lambda s: b"Correct" in s.posix.dumps(1),
              avoid=lambda s: b"Wrong"   in s.posix.dumps(1))
flag = simgr.found[0].posix.dumps(0)
```
```python
# argv input model
argv1 = claripy.BVS('argv1', 20 * 8)
state = proj.factory.entry_state(args=['./chall', argv1])
simgr = proj.factory.simgr(state)
simgr.explore(find=0x401337, avoid=0x401300)
print(simgr.found[0].solver.eval(argv1, cast_to=bytes))
```
```python
# Start mid-binary at the check function, inject a symbolic buffer directly
addr_check = 0x401200
state = proj.factory.blank_state(addr=addr_check)
buf = 0x600000
sym = claripy.BVS('inp', 16 * 8)
state.memory.store(buf, sym)
state.regs.rdi = buf                          # arg1 = pointer to input
simgr = proj.factory.simgr(state)
simgr.explore(find=0x401337, avoid=0x401300)
print(simgr.found[0].solver.eval(sym, cast_to=bytes))
```
```python
# --- Hooking to beat state explosion / skip expensive funcs ---
# Replace a whole library function with a SimProcedure:
class NotVeryRand(angr.SimProcedure):
    def run(self):
        return 0x1234
proj.hook_symbol('rand', NotVeryRand())

# Inline hook: skip a slow/irrelevant call and set a return value.
@proj.hook(0x401180, length=5)                # length = bytes of the patched call
def skip_check(state):
    state.regs.rax = 1                         # pretend it succeeded

# Optional: manage explosion
simgr.use_technique(angr.exploration_techniques.Veritesting())   # merge paths
# or add angr.options.LAZY_SOLVES to defer constraint solving
```

## Branching logic
- Multiple failure exits -> pass a LIST of avoid addresses: `avoid=[0x401300, 0x4013a0]`.
- Don't know the length -> try increasing lengths, or leave stdin unbounded and rely on output-based find; add a null/newline terminator constraint.
- Path count explodes / runs out of RAM -> add more avoids, hook the heavy function, enable Veritesting, or start at the check function (blank_state) to skip setup.
- Uses `scanf("%d")`/atoi -> the symbolic transform through libc parsing can bloat; hook scanf/atoi with a SimProcedure returning your symbolic var.
- Strong crypto (hash/AES) in the check -> angr will NOT solve it; recover the constant and invert manually or use z3 on the extracted equations.
- PIE binary -> load at a known base (`main_opts={'base_addr':0}`) and compute find/avoid as base+offset, or use output-based find.
- Reads from a file, not stdin -> back a `SimFile` with the symbolic bytes and register it in `fs`.

## Common pitfalls
- Wrong find/avoid addresses (off by PIE base, or pointing at the wrong block) -> empty `found`. Verify addresses against Ghidra/objdump and PIE status.
- Forgetting `auto_load_libs=False` -> angr traces real libc, slow and noisy.
- Not constraining the charset -> solver returns valid-but-nonprintable bytes that the real binary rejects; always constrain printable/length.
- Under- or over-long symbolic input vs what the program reads (missing newline the program's fgets expects) -> add the terminator.
- `posix.dumps(0)` returns ALL stdin consumed, which may include trailing padding — strip to the flag.
- Symbolic explosion from loops with symbolic bounds; keep input length concrete.

## Proof / verification
- Feed the recovered input back to the UNMODIFIED binary: `echo -n 'RECOVERED' | ./chall` must print success/the flag.
- If solving mid-function with blank_state, double-check the calling convention (RDI = arg1 on x86-64) so the pointer wiring is correct.
- Sanity-check the flag matches the challenge's flag format before submitting.

## Cross-references
- Get find/avoid addresses & the check's shape -> [sa--instructions--reverse-engineering--ghidra-workflow.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--ghidra-workflow.md)
- Confirm a candidate input live / when angr is overkill -> [sa--instructions--reverse-engineering--re-dynamic-gdb.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--re-dynamic-gdb.md)
- Fingerprint length/format/arch first -> [sa--instructions--reverse-engineering--re-triage-static.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--re-triage-static.md)
- Must unpack/deobfuscate before angr can load real code -> [sa--instructions--reverse-engineering--anti-debug-and-packing.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--anti-debug-and-packing.md)

## Credible sources
- angr docs — Simulation Managers (explore/find/avoid/stashes): https://docs.angr.io/en/latest/core-concepts/pathgroups.html
- angr docs — SimProcedures / hooking: https://docs.angr.io/en/latest/extending-angr/simprocedures.html
- angr install & getting started: https://docs.angr.io/en/latest/getting-started/installing.html
- angr_ctf exercises (jakespringer): https://github.com/jakespringer/angr_ctf
- picoCTF angr tutorial: https://picoctfsolutions.com/posts/angr-ctf-tutorial
