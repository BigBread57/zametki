# Vulnerability catalog

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / knowledge/vuln-catalog.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/knowledge/vuln-catalog.md)

---

# Vulnerability catalog

Cross-category lookup table of common CTF bug patterns and their tells.
**Check this table before reasoning from scratch** — if a recon signal
matches a row with a Template, go straight to it instead of exploring.
This is the fast path; the matching category skill is the fallback for
anything that doesn't hit a row here.

Rows with a **Template** are proven — confirmed working against a real
solve in this repo, with a parameterized, reusable version in `exploits/`.
Rows without one are either too bespoke to template (e.g. a custom VM's
specific bug) or unconfirmed (marked explicitly) — still worth the lookup,
but treat as a starting hypothesis, not a known-working recipe.

| Tell | Likely bug | Category | First move | Template |
|---|---|---|---|---|
| GraphQL endpoint, introspection enabled, an `id`-taking resolver returns another user's full record | Broken object-level auth (IDOR/BOLA) on a resolver | web | Dump `__schema`, scan for privileged bare queries + PII/token fields returned per-id | [na--exploits--graphql-idor-privileged-field.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/na--exploits--graphql-idor-privileged-field.md) |
| Menu-driven heap service (create/edit/delete/read) where delete doesn't null the pointer/size bookkeeping | Use-after-free → tcache poisoning | pwn | `delete(i); read(i)` — does it still return data? If yes, this is it | `exploits/heap-uaf-tcache-poison.py` |
| Action described as "queued"/"batched"/settled asynchronously (withdrawals, orders, transfers, redemptions) | Check-then-act (TOCTOU) race condition — validation runs at request time, mutation deferred | web | Fire N concurrent requests against the same un-mutated state | `exploits/toctou-race-condition.sh` |
| `file` reports generic "data" but `binwalk` finds a filesystem signature (SquashFS/JFFS2/cramfs) at some offset | Embedded-Linux/router firmware dump | forensics | `binwalk -e`, grep extracted tree for `flag`/`secret`/`*.bak`/`*config*` | `exploits/firmware-extract-grep-secrets.sh` |
| `{{var}}` interpolation syntax, both template string AND context are user-controlled ("preview" style feature) | Server-side template injection (SSTI) — likely needs an AST/compiler-level payload, not just `{{7*7}}`, if it's Handlebars | web | Try a naive expression probe first; if the engine sandboxes `constructor` access, escalate to a compiler-gadget payload | [na--exploits--handlebars-ssti-payloads.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/na--exploits--handlebars-ssti-payloads.md) *(unconfirmed — see file)* |
| Java + `jackson-databind`, `@JsonTypeInfo` + `PolymorphicTypeValidator` allow-list present, AND a bean setter somewhere has a side effect (spawns a process, opens a file) | Deserialization RCE via setter-side-effect gadget, likely bypassing the allow-list through a concrete-typed nested field the validator never checks | web / misc | Grep every setter for a side effect, then check whether it's reachable through a field that's concretely typed (not `Object`/polymorphic) inside an already-allowed class | [na--exploits--jackson-deserialization-setter-gadget.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/na--exploits--jackson-deserialization-setter-gadget.md) *(methodology only, unconfirmed)* |
| Custom bytecode VM / interpreter: an index/offset bound-check happens on the *base* register value, then an attacker-controlled offset is added *after* the check | Post-check OOB read/write (check-then-add-index bug class) | pwn / reverse | Find every place a checked index feeds into pointer arithmetic — look specifically for the check happening before, not after, the final address computation | none — too bespoke per-VM; see `writeups/published/UCSI26-sandworm.md` for a full worked example |
| `md5($a) == $b` / loose `==` on user input | PHP type juggling | web | | |
| `__proto__`/`constructor` accepted in a JSON body | Prototype pollution | web | | |
| Serialized-object magic bytes (`O:`, `\xac\xed`) in a cookie/param | Insecure deserialization | web | | |
| JWT `alg: none` accepted, or HS/RS confusion possible | JWT forgery | web | | |
| Small RSA `e` (3) with no padding | Cube-root attack | crypto | | |
| Two RSA ciphertexts, same `n`, different `e` | Common-modulus attack | crypto | | |
| RSA `d` small relative to `n` | Wiener's attack | crypto | | |
| `p`, `q` numerically close | Fermat factorization | crypto | | |
| Repeated 16-byte ciphertext blocks | ECB mode | crypto | | |
| Server distinguishes padding errors from other errors | CBC padding oracle | crypto | | |
| `printf(user_input)` without a format string literal | Format-string bug | pwn | | |
| Fixed-size buffer + `gets`/unbounded `scanf`/`strcpy` from larger input | Stack buffer overflow | pwn | | |
| No canary, no NX | Shellcode-on-stack is viable | pwn | | |
| All mitigations on, but a leak primitive exists (e.g. `puts(buf)`) | Leak-then-ROP | pwn | | |
| Heap overflow into an adjacent chunk that holds a callback/function pointer, no PIE | Direct callback-pointer overwrite (simpler than full heap exploitation — check chunk adjacency before reaching for tcache tricks) | pwn | Map allocation order and chunk sizes first; if a small fixed-size buffer sits directly before a struct with a function pointer, this may be all you need | none — see `challenges/picoCTF2026/heap-havoc/notes.md` for a worked 32-bit example |
| Very few strings + high section entropy | Packed binary | reverse / malware | | |
| `strcmp`/`memcmp` against a literal deep in disassembly | Hardcoded check — look for the literal directly | reverse | | |
| Repeated 16-byte pattern or high entropy region in a media file | Appended/embedded second file | forensics / stego | | |
| `exiftool` Warning: "Trailer data after ... IEND/EOF" (any container format) | Data appended past the file's logical end — check even if the Comment field looks like a deliberate red herring | stego / forensics | | |
| GPS/author metadata present on a provided image | Direct OSINT/forensics lead — check before anything else | forensics / osint | | |
| Padding error / oracle-like timing difference on a network service | Padding or timing side-channel | crypto / pwn | | |

## Maintenance

Every time a challenge is solved, check whether its bug pattern is already
a row here. If not, add one. If the same technique gets used a second
time, promote the exploit code to `exploits/` (parameterized, not the raw
one-off script) and link it from the Template column — that's what turns
this table from a reference into a fast path.


---

## Зависимости и ссылки вне библиотеки

Следующие файлы или ресурсы упомянуты в исходнике, но не входят в эту библиотеку документов. Их наличие и работоспособность здесь не проверялись.

- `exploits/firmware-extract-grep-secrets.sh`
- `exploits/heap-uaf-tcache-poison.py`
- `exploits/toctou-race-condition.sh`
