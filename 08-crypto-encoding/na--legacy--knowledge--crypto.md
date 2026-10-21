# crypto

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.
> Архивная методика старой реализации; упомянутые функции и обёртки в библиотеку не включены.

Источники этого текста:

- [nonoleekr/CTF-Agent / archive/legacy-python-agent/knowledge/crypto.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/archive/legacy-python-agent/knowledge/crypto.md)

---

Crypto heuristics:

- Check whether the *entire* ciphertext (not just part of it) is valid
  base64 or hex first — `crypto_analyze` only recognizes a full match, so
  a ciphertext embedded in surrounding prose will not be detected as such.
- After decoding, try ROT13/ROT-N and single-byte XOR before anything more
  elaborate — both are automated by `crypto_analyze` and catch a large
  share of simple substitution/obfuscation challenges.
- If those fail, look for a repeating-key XOR pattern, a classical cipher
  (Caesar, Vigenère, Atbash), or a weak/misused hash (unsalted MD5/SHA1
  reversible via a lookup, length-extension-vulnerable constructions).
- For RSA-flavored challenges, check for a small modulus, shared/reused
  primes across keys, or a small public exponent before assuming a strong
  key — none of this is automated yet; it's a manual next step.
- Treat every decode/rotation as a hypothesis, not a conclusion — confirm
  the result actually looks like a flag or readable text before trusting it.
