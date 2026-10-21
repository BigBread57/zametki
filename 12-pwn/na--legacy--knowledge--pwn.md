# pwn

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.
> Архивная методика старой реализации; упомянутые функции и обёртки в библиотеку не включены.

Источники этого текста:

- [nonoleekr/CTF-Agent / archive/legacy-python-agent/knowledge/pwn.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/archive/legacy-python-agent/knowledge/pwn.md)

---

Pwn heuristics:

- Check the binary's security mitigations first — `pwn_analyze` now
  natively parses NX, PIE, and RELRO from the ELF headers, plus a
  `__stack_chk_fail` symbol heuristic for a canary. What's disabled tells
  you what's exploitable: no NX means shellcode-on-the-stack is viable, no
  PIE means addresses are fixed and easier to target.
- Look for the actual input-handling function (`gets`, unbounded `scanf`,
  a fixed-size `read`/`strcpy` into a smaller buffer) to find the overflow
  point, and note the offset needed to reach the return address or a
  saved variable.
- A format-string bug (user input passed directly as a `printf` format)
  is a separate, often easier win — check for `%x`/`%n`/`%s` reachable
  from attacker-controlled input before assuming you need a full overflow.
- With no canary and no NX, a straightforward stack-smashing shellcode
  payload is often enough; with mitigations on, look toward ROP using the
  binary's own gadgets (`ROPgadget`/`objdump` — not automated here yet).
- Confirm any candidate exploit against the actual binary/service before
  trusting it — a plausible-looking offset that isn't verified is a guess.
