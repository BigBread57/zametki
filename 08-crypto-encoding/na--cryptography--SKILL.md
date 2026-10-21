# Cryptography

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / .claude/skills/cryptography/SKILL.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/.claude/skills/cryptography/SKILL.md)

---

---
name: cryptography
description: Playbook for cryptography challenges — ciphertext, an RSA public key/modulus, a hash to crack, encoded-looking text, or a description mentioning encrypt/decrypt/cipher/key. Use after ctf-methodology has classified the challenge as crypto.
---

# Cryptography

Reference commands: [na--knowledge--tools-reference--cryptography.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/na--knowledge--tools-reference--cryptography.md). Attack
conditions table: [na--knowledge--vuln-catalog.md](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/na--knowledge--vuln-catalog.md). Script skeletons:
[na--knowledge--exploit-patterns.md](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/na--knowledge--exploit-patterns.md).

## Approach

1. **Classify what you're actually looking at** before reaching for an attack:
   - Encoded, not encrypted (base64/hex/rot13) — decode, don't attack.
   - Classical cipher (substitution, Vigenère) — solvable by frequency
     analysis or a known-plaintext crib.
   - Modern primitive used correctly — look for an implementation mistake
     around it (key reuse, weak randomness, side channel), not a break of the
     primitive itself.
   - Modern primitive used incorrectly — the bug is in the misuse, check
     [na--knowledge--vuln-catalog.md](https://raw.githubusercontent.com/BigBread57/zametki/main/00-methodology/na--knowledge--vuln-catalog.md) for the specific tell (small RSA `e`, ECB
     repetition, CBC oracle, etc.) and go straight to the matching attack.
2. **Chained encodings**: decode one layer, look at the result — is it
   plaintext, another encoding, or binary — before assuming you're stuck or
   reaching for a tool. Most "weird ciphertext" is just base64-of-hex or
   similar, solvable in one line.
3. **RSA**: run `.venv/bin/factordb <n>` before anything else — many CTF moduli
   are already factored publicly. If that fails, check the conditions in
   order: small `e` → cube-root attack, shared `n` across ciphertexts →
   common-modulus, small `d` → Wiener's, `p`/`q` close → Fermat.
4. **XOR**: single-byte key → brute force all 256 values and eyeball for
   readable output. Repeating-key → estimate key length via
   Hamming-distance/index-of-coincidence, then solve each column as
   single-byte XOR.
5. **Hashes**: identify the algorithm (length/format), then `hashcat`/`john`
   against `wordlists/Passwords/rockyou.txt` before assuming it needs to be
   broken cryptographically — most CTF "crack this hash" challenges are
   dictionary-crackable.

## Verification discipline

Don't submit a decryption that merely looks plausible (readable-ish text) —
confirm it matches the challenge's expected flag format exactly, since a wrong
key or off-by-one in a manual attack often produces near-readable garbage.
