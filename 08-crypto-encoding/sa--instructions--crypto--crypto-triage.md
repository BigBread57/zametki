# Crypto Triage & Routing

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/crypto/crypto-triage.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/crypto/crypto-triage.md)

---

---
name: crypto-triage
category: crypto
triggers: ["unknown ciphertext", "base64", "hex", "decimal blob", "encoding", "what cipher", "gibberish string", "==", "0x", "identify crypto", "CyberChef", "magic"]
tools: ["CyberChef", "dcode.fr", "python", "hashid", "file", "xxd", "strings"]
priority: high
description: First-look triage that sorts an unknown crypto challenge into encoding vs classical cipher vs modern crypto and routes to the right playbook.
---
# Crypto Triage & Routing

## Trigger context
You are handed an opaque blob, a `.py`/`.sage` challenge script, a public key, or a
network capture and you do not yet know whether it is encoding, a classical cipher,
or modern crypto (RSA/AES/ECC/hash). Start here before committing to any attack.

## Objective
Classify the artifact in under a minute, recover its true byte content from any
encoding layers, and jump to the correct specialized file. Encoding is NOT
encryption: it carries no key and is always fully reversible.

## Methodology (decision tree)
1. **Read the source first.** If a `.py`/`.sage`/`.txt` script is provided, read it.
   The variable names reveal everything: `n, e, c` => RSA, `AES.new(key, MODE_ECB)`
   => AES ECB, `key * ...` / `^` over bytes => XOR, `p, g, A, B` => Diffie-Hellman,
   `E = EllipticCurve` => ECC. Named parameters beat guessing every time.
2. **Character-set fingerprint** of the blob:
   - Only `[0-9a-fA-F]`, even length => hex. Decode: `bytes.fromhex()`.
   - `[A-Za-z0-9+/]` with `=` padding, len % 4 == 0 => Base64. `+/` => standard,
     `-_` => URL-safe, uppercase-only w/ `=` => Base32.
   - All decimal digits, one huge number => could be a big-int ciphertext (RSA `c`)
     or decimal-encoded bytes. Try `long_to_bytes(int(x))`.
   - `[A-Z2-7]` + `=` => Base32. `[0-9A-Za-z]` no symbols, ~1.3x => Base58/Base62.
   - Starts `{` / ends `}`, or contains `_`, letters+digits mixed => maybe the flag
     already (check for the flag format, e.g. `flag{`, `CTF{`).
3. **Decode layers greedily.** Encodings nest (base64(hex(...))). Keep decoding while
   output still looks like a known charset. Use CyberChef "Magic" or the loop below.
4. **After decoding, judge the plaintext-ness:**
   - Printable ASCII / recognizable words => done, or a classical cipher => [sa--instructions--crypto--xor-and-classical.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--xor-and-classical.md).
   - High-entropy random bytes, length a multiple of 8 or 16 => block cipher =>
     [sa--instructions--crypto--aes-block-cipher.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--aes-block-cipher.md). Length 16/24/32 also hints AES key sizes.
   - One giant integer alongside a modulus/exponent => [sa--instructions--crypto--rsa-attacks.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--rsa-attacks.md).
   - Fixed-length hex 32/40/64/128 or `$2b$`/`$6$` => a hash => [sa--instructions--crypto--hash-cracking.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--hash-cracking.md).
5. **Entropy check** distinguishes cipher from encoding: `ent`/Python Shannon entropy
   near 8 bits/byte => encrypted or compressed; low entropy => text or weak cipher.

## Commands & payloads
```python
# Universal decode-loop triage helper
import base64, binascii, re, math
from collections import Counter

def entropy(b):
    if not b: return 0
    c = Counter(b); n = len(b)
    return -sum((v/n) * math.log2(v/n) for v in c.values())

def try_all(s):
    s = s.strip()
    out = {}
    # hex
    if re.fullmatch(r'[0-9a-fA-F]+', s) and len(s) % 2 == 0:
        out['hex'] = binascii.unhexlify(s)
    # base64 (std + urlsafe)
    for name, fn in [('b64', base64.b64decode), ('b64url', base64.urlsafe_b64decode)]:
        try: out[name] = fn(s + '=' * (-len(s) % 4))
        except Exception: pass
    try: out['b32'] = base64.b32decode(s.upper() + '=' * (-len(s) % 8))
    except Exception: pass
    # decimal big-int -> bytes
    if s.isdigit():
        h = format(int(s), 'x'); h = ('0'+h) if len(h) % 2 else h
        out['decimal'] = binascii.unhexlify(h)
    return out

for k, v in try_all(open('blob.txt').read()).items():
    print(f"{k:8} entropy={entropy(v):.2f}  {v[:48]!r}")
```
```bash
file mystery.bin          # magic-byte ID: PNG/ZIP/PEM/etc hiding inside
strings -n 8 mystery.bin  # look for flag fragments / PEM markers
xxd mystery.bin | head     # inspect raw structure, block boundaries
hashid -m 'deadbeef...'   # if it looks like a hash, get hashcat/john modes
```

## Quick format signature table
| Signature | Likely type | Route |
|-----------|-------------|-------|
| `[0-9a-f]` even len | hex encoding | decode, re-triage |
| `[A-Za-z0-9+/]` + `=` | Base64 | decode, re-triage |
| `[A-Z2-7]` + `=` | Base32 | decode, re-triage |
| `-----BEGIN ... KEY-----` | PEM key | [sa--instructions--crypto--rsa-attacks.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--rsa-attacks.md) |
| one huge integer | RSA `c` / big-int bytes | [sa--instructions--crypto--rsa-attacks.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--rsa-attacks.md) |
| 32/40/64/128 hex, fixed | hash digest | [sa--instructions--crypto--hash-cracking.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--hash-cracking.md) |
| `$2b$` / `$6$` / `$1$` | crypt hash | [sa--instructions--crypto--hash-cracking.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--hash-cracking.md) |
| len % 16 == 0, high entropy | AES block cipher | [sa--instructions--crypto--aes-block-cipher.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--aes-block-cipher.md) |
| letters shifted / A-Z only | classical cipher | [sa--instructions--crypto--xor-and-classical.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--xor-and-classical.md) |
| `.` `-` groups | Morse | dcode.fr |
| `Ook.` / `+++.` | Brainfuck/esolang | dcode.fr / interpreter |

## Branching logic
- Decoded to readable text but shifted/scrambled -> [sa--instructions--crypto--xor-and-classical.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--xor-and-classical.md).
- `n`, `e`, `c` present, or a `.pem`/`BEGIN PUBLIC KEY` -> [sa--instructions--crypto--rsa-attacks.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--rsa-attacks.md).
- Ciphertext length % 16 == 0, AES/DES in source, IV/nonce present -> [sa--instructions--crypto--aes-block-cipher.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--aes-block-cipher.md).
- Fixed-width hex digest or `$id$salt$hash` crypt string -> [sa--instructions--crypto--hash-cracking.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--hash-cracking.md).
- ECC / DLP / lattice / unusual math -> beyond these files; note params and research.

## Common pitfalls
- Treating Base64 as "encryption": there is no secret, do not brute-force it.
- Missing nested encodings; always re-run Magic on the decoded output.
- Confusing hex (charset 0-9a-f) with Base64; hex has no uppercase-vs-symbol mix.
- Assuming `=` means Base64 only, Base32 also pads with `=`.
- Ignoring the provided source script and reverse-engineering the format from bytes.

## Proof / verification
You have triaged correctly when the decoded bytes either read as plaintext/flag or
match the structural signature (block length, integer size, digest width) that the
target playbook expects. Confirm by feeding the recovered bytes into that file's
first sanity check.

## Cross-references
- [sa--instructions--crypto--rsa-attacks.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--rsa-attacks.md), [sa--instructions--crypto--aes-block-cipher.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--aes-block-cipher.md), [sa--instructions--crypto--xor-and-classical.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--xor-and-classical.md), [sa--instructions--crypto--hash-cracking.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--hash-cracking.md).

## Credible sources
- CyberChef (Magic operation): https://gchq.github.io/CyberChef/
- dcode.fr cipher identifier: https://www.dcode.fr/cipher-identifier
- CTF101 Cryptography overview: https://ctf101.org/cryptography/overview/
- HackTricks crypto index: https://book.hacktricks.wiki/en/crypto-and-stego/index.html
