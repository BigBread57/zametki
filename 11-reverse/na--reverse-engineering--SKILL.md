# Reverse engineering

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / .claude/skills/reverse-engineering/SKILL.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/.claude/skills/reverse-engineering/SKILL.md)

---

---
name: reverse-engineering
description: Playbook for reverse-engineering challenges — an ELF/PE binary with no network target, a "what does this program do" or "find the hidden logic" framing, or an obfuscated/packed sample to analyze statically. Use after ctf-methodology has classified the challenge as reverse (not pwn — see binary-exploitation for exploitation-focused binaries).
---

# Reverse engineering

Reference commands: [na--knowledge--tools-reference--reverse-engineering.md](https://raw.githubusercontent.com/BigBread57/zametki/main/11-reverse/na--knowledge--tools-reference--reverse-engineering.md).

## Approach

1. **Triage first**: `file`, `checksec --file=`, `strings -n 8`. Stripped?
   Statically or dynamically linked? Any obvious version/library banners in
   the strings output?
2. **Static read**: prefer the `radare2` MCP tool (configured in `.mcp.json`)
   over driving `r2`'s interactive shell — `open_file` → `analyze` →
   `list_functions` → `decompile_function`/`disassemble_function` on anything
   interesting gives structured, parseable results instead of scraping
   terminal output each step. Look for anything not named `sub_...`/`fcn.*`
   first (unstripped symbols are free information), then `main` and anything
   it calls early. `decompile_function` uses r2's `pdc` backend — good enough
   for most CTF binaries; fall back to Ghidra headless (`analyzeHeadless`) for
   anything genuinely complex where a real decompiler's output would help
   more than r2's pseudo-C.
3. **Decide static vs. dynamic.** Switch to `gdb` (pwndbg auto-loads) as soon
   as static reading would require guessing about a runtime value you could
   just observe — a computed jump table target, a string decrypted at
   startup, an unknown loop bound tied to input length.
4. **Packing/obfuscation check**: very few strings + high-entropy section →
   likely packed. `strings | grep -i upx` and `upx -d` if UPX; otherwise note
   it and expect a dynamic-analysis step to reveal the real logic after
   unpacking happens at runtime.
5. **Symbolic execution when static+dynamic reading isn't converging** — e.g.
   "what input reaches this specific branch/address." Use the `angr` MCP tool
   (configured in `.mcp.json`) rather than hand-writing an angr script each
   time; it exposes CFG recovery and path exploration as structured calls.

## What "solved" looks like

Either the flag is a string revealed by correct execution/input (confirm by
actually running the binary with that input if possible), or it's derivable
directly from the logic you've read (e.g. a hardcoded comparison value) — in
the second case, double-check you've read the comparison correctly rather than
transcribing a disassembly typo into the submitted flag.


---

## Зависимости и ссылки вне библиотеки

Следующие файлы или ресурсы упомянуты в исходнике, но не входят в эту библиотеку документов. Их наличие и работоспособность здесь не проверялись.

- `.mcp.json`
