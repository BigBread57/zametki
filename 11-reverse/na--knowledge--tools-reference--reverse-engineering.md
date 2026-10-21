# Reverse engineering — tools reference

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / knowledge/tools-reference/reverse-engineering.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/knowledge/tools-reference/reverse-engineering.md)

---

# Reverse engineering — tools reference

## First pass (always, before opening a disassembler)
```
file <binary>                          # arch, static/dynamic linking, stripped?
.venv/bin/checksec --file=<binary>      # mitigations — also tells you a lot about how it was built
strings -n 8 <binary> | less            # look for format strings, error messages, version banners
```

## radare2
```
r2 -A <binary>          # -A = auto-analyze on load
aaa                      # (if not auto-analyzed) full analysis
afl                       # list functions
pdf @ main                  # disassemble a function
axt @ <addr>                 # cross-refs to an address
s main; V                     # seek to main, open visual mode
```
`r2pipe` (in `.venv/`) lets you script a session instead of driving it
interactively — worth it once you're repeating the same few commands across
several functions.

## Ghidra (headless, no GUI needed)
```
~/tools/ghidra_12.1.2_PUBLIC/support/analyzeHeadless <project-dir> <project-name> \
  -import <binary> -postScript <script.py> -deleteProject
```
Use headless mode with a post-script for anything that would otherwise require
clicking through the GUI. Reach for the GUI only if the user is going to look at
it themselves.

## gdb + pwndbg (already auto-loaded via ~/.gdbinit)
```
gdb <binary>
b *main            # break at main
r                  # run
x/20i $pc          # disassemble at current PC
info functions       # symbol list, if not stripped
```

## angr (symbolic execution — via `angr-mcp` MCP tool or `.venv/bin/python`)
Use when static reading isn't enough to determine what input reaches a target
branch/address — e.g. "what input makes this function return 1" or "find a path
to the win() function." Prefer the `angr-mcp` MCP tool (configured in
`.mcp.json`) over hand-writing angr scripts each time; it exposes CFG recovery
and path exploration as structured calls.

## Static vs. dynamic — when to switch
Start static (strings, disassembly, decompilation). Switch to dynamic
(gdb/ltrace/strace) as soon as static reading requires guessing about runtime
values you could just observe — e.g. a computed jump table, a
runtime-decrypted string, an unknown loop bound.

## Packing / obfuscation tells
- Very few strings + high entropy in a large section → likely packed; check for
  UPX (`strings <binary> | grep -i upx`, then `upx -d` to unpack if present).
- A tiny number of imports for a binary that clearly does a lot → possible manual
  unpacking stub or static linking worth investigating.


---

## Зависимости и ссылки вне библиотеки

Следующие файлы или ресурсы упомянуты в исходнике, но не входят в эту библиотеку документов. Их наличие и работоспособность здесь не проверялись.

- `.mcp.json`
