# reverse

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.
> Архивная методика старой реализации; упомянутые функции и обёртки в библиотеку не включены.

Источники этого текста:

- [nonoleekr/CTF-Agent / archive/legacy-python-agent/knowledge/reverse.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/archive/legacy-python-agent/knowledge/reverse.md)

---

Reverse-engineering heuristics:

- Start with file type — `reverse_analyze` distinguishes ELF, PE, and
  shebang scripts by magic bytes, and a script is usually far faster to
  read than a compiled binary.
- Printable strings are the cheapest source of clues: format strings,
  error messages, hardcoded comparison values, and occasionally the flag
  itself all show up in a plain strings scan before any disassembly.
- For a script, just read it — obfuscation in CTF scripts is usually
  shallow (base64/eval chains, string reversal, simple XOR) and rarely
  needs a real deobfuscator.
- For a compiled binary, look for the comparison against user input (the
  "check" function) rather than reading the whole program — that's almost
  always where the logic that gates the flag lives.
- Static analysis (strings, imports) only gets you so far; real
  disassembly/decompilation (Ghidra, objdump) is the next step for
  anything beyond a simple string-compare check, and isn't automated here
  yet.
