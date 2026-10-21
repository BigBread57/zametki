# Ghidra Workflow

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/reverse-engineering/ghidra-workflow.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/reverse-engineering/ghidra-workflow.md)

---

---
name: ghidra-workflow
category: reverse-engineering
triggers: ["ghidra", "decompiler", "decompile", "pseudo-C", "rename variable", "retype", "find main", "strcmp", "flag logic", "analyzeHeadless", "ghidra script", "stripped binary"]
tools: [Ghidra, "Ghidra decompiler", analyzeHeadless, "Ghidra Python/Jython", PyGhidra, ghidra_scripts]
priority: high
description: Import, auto-analyze, and read a binary in Ghidra's decompiler to follow flag logic, plus rename/retype and headless scripting.
---
# Ghidra Workflow

## Trigger context
The logic is non-trivial and you need readable pseudo-C to follow how input becomes accept/reject. Ghidra is the free, batteries-included choice: multi-arch decompiler + scripting. Use it when strings/ltrace didn't hand you the flag and the control flow needs to be understood, not just forced.

## Objective
- Get from raw bytes to annotated pseudo-C for the check/flag routine.
- Locate `main`/the validator, follow the comparison, and recover the expected value or the transform.
- Optionally automate extraction with headless scripts.

## Methodology (decision tree)
1. Import (`I` or File -> Import File). Confirm the detected language/arch matches triage; override if wrong.
2. Auto-analyze with defaults (accept the analyzer prompt). For Go binaries enable the Golang analyzer; for large/static, enable Decompiler Parameter ID.
3. Find `main`:
   - Not stripped: Symbol Tree -> Filter "main".
   - Stripped: open `entry`, find the call to `__libc_start_main`; on x86-64 its first arg (loaded into RDI just before the call) is `main`. Double-click it.
4. Open the Decompiler (double-click function). Read top-down.
5. Follow the flag logic: look for `strcmp/strncmp/memcmp`, `==`, loops that XOR/add/index a buffer, or a `local_XX` compared to a constant.
6. Annotate as you go: rename variables/functions (`L`), retype (`Ctrl+L` for var type, edit function signature), set an array/char type so the decompiler shows strings.
7. Extract the expected value (a constant, a byte array, or an inverse of the transform). Solve the inverse or hand a hard constraint to angr.
8. Repeat for helper functions; use the Function Call Trees / References (`Ctrl+Shift+F` = find references to a symbol).

## Commands & payloads
```bash
# --- Headless (batch) analysis: import + run a script, no GUI ---
# Linux: $GHIDRA_HOME/support/analyzeHeadless   Windows: analyzeHeadless.bat
analyzeHeadless /path/to/projDir ProjName \
    -import ./chall \
    -scriptPath ./ghidra_scripts \
    -postScript ExportDecomp.py \
    -deleteProject          # discard project after (omit to keep)

# analyze an already-imported program instead of -import:
analyzeHeadless /path/to/projDir ProjName -process chall -postScript ExportDecomp.py

# PyGhidra (Ghidra 11+, real CPython) one-off decompile of a binary
pyghidra ./chall            # or use pyghidra.open_program() in a script
```
```python
# ExportDecomp.py  -- a Ghidra script (Jython API; run as postScript or in Script Manager)
# Dumps pseudo-C for every function; grep the output for 'strcmp'/'flag'.
from ghidra.app.decompiler import DecompInterface
from ghidra.util.task import ConsoleTaskMonitor

program = getCurrentProgram()          # provided by Ghidra script context
ifc = DecompInterface()
ifc.openProgram(program)
fm = program.getFunctionManager()
for func in fm.getFunctions(True):      # True = forward order
    res = ifc.decompileFunction(func, 60, ConsoleTaskMonitor())
    if res.decompileCompleted():
        print("// ==== %s @ %s ====" % (func.getName(), func.getEntryPoint()))
        print(res.getDecompiledFunction().getC())
```
```python
# FindStrcmpCallers.py -- list call sites of strcmp/strncmp/memcmp with the constant arg
fm = getCurrentProgram().getFunctionManager()
st = getCurrentProgram().getSymbolTable()
targets = ("strcmp", "strncmp", "memcmp")
for sym in st.getAllSymbols(True):
    if sym.getName() in targets:
        for ref in sym.getReferences():
            frm = ref.getFromAddress()
            caller = fm.getFunctionContaining(frm)
            print("%s called from %s @ %s" % (sym.getName(),
                  caller.getName() if caller else "?", frm))
```

## Branching logic
- Decompiler shows `strcmp(input, "S3CR3T")` -> that literal is the answer.
- Shows a loop like `for i: out[i] = in[i] ^ key[i]; if(out != target) fail` -> XOR `target` with `key` to recover input; if key/target are byte arrays, dump them (right-click -> Copy Special / read from the Listing).
- Constant compared but produced by a function -> rename+read that function; if it is a self-contained deterministic checker, stop reversing and route the accept/reject addresses to angr.
- Stripped + no libc names -> the analyzer may still tag `strcmp` via signatures; if not, recognize it by shape (null-terminated compare loop).
- Wrong arch/garbage decompile -> re-import with the correct Language ID; check endianness.
- Go/Rust mangling -> apply the Golang analyzer / use symbol demangling; function names may live in a `.gopclntab`.

## Common pitfalls
- Accepting auto-analysis defaults blindly on a stripped static binary can be slow; still let it finish before reading.
- The decompiler invents `local_XX`/`uVar` names and sometimes wrong types (int vs char*, missed arrays); retype to make strings/loops legible before trusting it.
- Ghidra's default P-code may fold constants oddly; cross-check a suspicious constant against the raw Listing/disassembly.
- Off-by-one: decompiled index math can hide a `+1`/length check that matters for the flag length.
- Headless scripts run with the Jython API (Ghidra <=10 / bundled), while PyGhidra uses CPython — API objects are the same, imports/launcher differ.
- Editing a function signature changes the decompilation everywhere it's called — good, but verify you didn't mistype the calling convention.

## Proof / verification
- Reconstruct the expected input from the decompiled transform and feed it to the real binary; it should print success/flag.
- Compare Ghidra's constant against the same address in the Listing view and, if possible, against a gdb runtime read.
- For headless extraction, grep the decompiled dump for `flag`, `strcmp`, and the flag prefix to confirm coverage.

## Cross-references
- Verify a decompiled constant at runtime / bypass live -> [sa--instructions--reverse-engineering--re-dynamic-gdb.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--re-dynamic-gdb.md)
- Deterministic checker with a hard transform -> [sa--instructions--reverse-engineering--angr-symbolic-execution.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--angr-symbolic-execution.md)
- Language/packer/arch fingerprint before import -> [sa--instructions--reverse-engineering--re-triage-static.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--re-triage-static.md)
- Binary is packed/anti-analysis before it decompiles cleanly -> [sa--instructions--reverse-engineering--anti-debug-and-packing.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--anti-debug-and-packing.md)

## Credible sources
- Ghidra project & docs: https://ghidra-sre.org/
- Ghidra GitHub (headless & scripting docs): https://github.com/NationalSecurityAgency/ghidra
- Ghidra API (FlatProgramAPI / DecompInterface): https://ghidra.re/ghidra_docs/api/
- PyGhidra: https://github.com/NationalSecurityAgency/ghidra/tree/master/Ghidra/Features/PyGhidra
- picoCTF Ghidra tutorial: https://picoctfsolutions.com/posts/ghidra-reverse-engineering
