# RE Static Triage

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/reverse-engineering/re-triage-static.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/reverse-engineering/re-triage-static.md)

---

---
name: re-triage-static
category: reverse-engineering
triggers: ["ELF binary", "PE binary", "Mach-O", "strings", "unknown binary", "file command", "checksec", "detect language", "detect packer", "first look", "quick win", "readelf", "objdump", "nm"]
tools: [file, strings, nm, objdump, readelf, checksec, rabin2, binwalk, xxd, ltrace, upx, "die (Detect It Easy)"]
priority: high
description: First-pass static triage of an unknown binary to fingerprint it, harvest low-hanging flags, and decide the next stage.
---
# RE Static Triage

## Trigger context
You are handed an unknown executable (ELF/PE/Mach-O) or a "reverse this" challenge and have not yet opened a disassembler. Always run this stage FIRST: it is fast, non-destructive, and frequently ends the challenge before any deep analysis.

## Objective
- Fingerprint the file: format, architecture, bitness, linkage (static/dynamic), stripped or not, source language, packer/obfuscation.
- Extract cheap wins: hardcoded flags, comparison strings, embedded files, format-string hints.
- Produce a routing decision: which next instruction file applies (dynamic-gdb, ghidra-workflow, anti-debug-and-packing, angr).

## Methodology (decision tree)
1. `file` -> is it ELF/PE/Mach-O, 32 vs 64-bit, statically vs dynamically linked, stripped? Non-executable? -> treat as forensics/binwalk target.
2. `strings` (and `strings -e l` for UTF-16 in PE) -> grep for the flag format (e.g. `flag{`, `CTF{`, `picoCTF{`). If present, you may be done.
3. If strings look compressed/high-entropy and section names are `UPX0/UPX1` or few imports -> go to anti-debug-and-packing (unpack first, then re-triage the unpacked file).
4. Detect language: symbol/string patterns tell you (see Branching logic). Go/Rust/.NET/Python-frozen need language-specific tooling, not plain Ghidra.
5. `checksec` -> note PIE/NX/RELRO/Canary. Relevant if this is really pwn; for pure RE it tells you if addresses are static (no PIE) so gdb offsets are stable.
6. `nm` / `readelf -s` (if not stripped) -> jump straight to interesting symbol names (`check`, `verify`, `validate`, `compare`, `secret`, `decrypt`).
7. `objdump -d` a quick scan of `main` for immediate `cmp`/`strcmp` against constants.
8. Still unsolved -> route: linear password/xor check -> angr; runtime checks/keys derived at runtime -> re-dynamic-gdb; complex logic -> ghidra-workflow.

## Commands & payloads
```bash
# --- fingerprint ---
file ./chall                      # format, arch, linkage, stripped?
rabin2 -I ./chall                 # radare2 info: canary, nx, pic, lang, stripped
readelf -h ./chall                # ELF header: class (ELF32/64), machine, entry
readelf -d ./chall                # dynamic section / shared-lib deps
checksec --file=./chall           # RELRO, canary, NX, PIE, RPATH (pwntools install)

# --- strings harvest ---
strings -n 6 ./chall | less
strings -n 6 ./chall | grep -Ei 'flag\{|ctf\{|pico|secret|passw|key|xor|base64'
strings -e l ./chall | grep -i flag        # -e l = 16-bit little-endian (PE/UTF-16)
strings -a -t x ./chall | grep flag        # -t x also prints file offset

# --- symbols & imports (only useful if NOT stripped) ---
nm ./chall 2>/dev/null | grep -Ei ' t | T '   # local/global text symbols (funcs)
nm -D ./chall                                 # dynamic symbols (imported libc funcs)
readelf -s ./chall | grep -i func
objdump -T ./chall                            # dynamic symbol table (PLT targets)

# --- quick disasm scan ---
objdump -d -M intel ./chall | grep -iA3 -E 'call.*strcmp|call.*strncmp|call.*memcmp|cmp '
objdump -d -M intel ./chall | less            # scan <main>

# --- embedded content / packing signal ---
binwalk ./chall                    # embedded archives, filesystems, keys
xxd ./chall | head                 # magic bytes by eye
readelf -S ./chall | grep -i upx   # UPX0/UPX1 section names => packed
upx -t ./chall                     # "OK" if UPX-packed
die ./chall                        # Detect It Easy: packer + compiler + lang (if installed)

# --- observe live library calls without a debugger ---
ltrace ./chall                     # reveals strcmp/strncmp args (often the password!)
strace -f ./chall                  # syscalls: file/env/ptrace access
```

## Branching logic
- `file` says "not stripped" + libc calls -> jump to Ghidra with symbols intact (easy).
- `file` says "statically linked, stripped" -> big binary; use FLIRT/signatures or focus on `main` found via entry-point tracing (see ghidra-workflow).
- Language fingerprints:
  - Go: strings contain `go:buildid`, `runtime.`, `GOROOT`, panic messages; use `GoReSym`, Ghidra Golang analyzer.
  - Rust: `rustc`, `panicked at`, `/cargo/registry`; symbols mangled `_ZN...`; c++filt/rustfilt.
  - .NET / C#: `mscoree`, PE with CLR header; use `dnSpy`/`ILSpy`, not Ghidra.
  - Python frozen: `PyInstaller`, `_MEIPASS`, `python3X.dll`; extract with `pyinstxtractor`, then `decompyle3`/`uncompyle6`.
  - C++: mangled `_Z...` names, `std::`; run through `c++filt`.
- `upx -t` == OK, or `UPX!` magic -> anti-debug-and-packing (unpack, then rerun this file).
- Found `flag{...}` in strings -> submit; if it is a decoy (many challenges plant fakes), continue.
- `strcmp`/`memcmp` against a static string near main -> that constant is likely the flag/password.

## Common pitfalls
- Flag split across bytes or XOR/base64-encoded won't show in `strings`; absence of a flag proves nothing.
- Stripped binaries have no `main` symbol; find it via `__libc_start_main`'s first arg (RDI on x86-64) at entry.
- `checksec`/PIE means runtime addresses are randomized; use offsets from the module base, not absolute addresses, in gdb.
- Running an unknown/malware-flavored binary can be hostile: prefer a VM/container; static triage first.
- `ltrace` fails on static binaries and is defeated by ptrace anti-debug; don't over-trust a blank ltrace.
- UTF-16 strings in PE need `strings -e l`; default `strings` misses them.

## Proof / verification
- Confirm arch/format with two tools (`file` and `readelf -h`/`rabin2 -I`) before committing to a toolchain.
- If you extract a candidate flag, verify by running the binary and feeding it back, or matching it against the checker's comparison constant.
- Record: format, arch, linkage, stripped?, language, packer?, checksec bits, and the chosen next stage.

## Cross-references
- Packed/UPX or anti-debug signals -> [sa--instructions--reverse-engineering--anti-debug-and-packing.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--anti-debug-and-packing.md)
- Need to run and watch comparisons live -> [sa--instructions--reverse-engineering--re-dynamic-gdb.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--re-dynamic-gdb.md)
- Need decompiled C to follow logic -> [sa--instructions--reverse-engineering--ghidra-workflow.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--ghidra-workflow.md)
- Deterministic input->accept checker -> [sa--instructions--reverse-engineering--angr-symbolic-execution.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/sa--instructions--reverse-engineering--angr-symbolic-execution.md)

## Credible sources
- CTF101 Reverse Engineering: https://ctf101.org/reverse-engineering/overview/
- HackTricks Reversing (basic methods): https://book.hacktricks.wiki/en/reversing/reversing-tools-basic-methods/index.html
- radare2 book (rabin2): https://book.rada.re/tools/rabin2/intro.html
- pwntools checksec: https://docs.pwntools.com/en/stable/commandline.html#pwn-checksec
- binwalk: https://github.com/ReFirmLabs/binwalk
