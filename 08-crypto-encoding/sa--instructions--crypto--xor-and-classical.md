# XOR & Classical Ciphers

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/crypto/xor-and-classical.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/crypto/xor-and-classical.md)

---

---
name: xor-and-classical
category: crypto
triggers: ["XOR", "single-byte XOR", "repeating key", "Vigenere", "crib dragging", "known plaintext", "two-time pad", "Caesar", "ROT13", "substitution", "rail fence", "frequency analysis"]
tools: ["python", "pycryptodome", "CyberChef", "dcode.fr", "xortool", "quipqiup"]
priority: high
description: Recover plaintext from XOR (single-byte, repeating-key, keystream reuse) and classical ciphers (Caesar, Vigenere, substitution, rail fence) via frequency analysis and crib dragging.
---
# XOR & Classical Ciphers

## Trigger context
Ciphertext that decodes to bytes but not plaintext: XOR against a key, a two-time pad
(reused keystream), or a paper cipher (Caesar/ROT, Vigenere, monoalphabetic
substitution, transposition/rail-fence). Often the last hop after [sa--instructions--crypto--crypto-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--crypto-triage.md).

## Objective
Recover the key and/or plaintext using statistical structure of English/flag text.
XOR and classical ciphers leak plaintext frequency; exploit it.

## Methodology (decision tree)
1. **Single-byte XOR:** brute force all 256 keys, score each candidate by English
   letter frequency (or count of printable/space chars). Best score = key.
2. **Repeating-key XOR (Vigenere over bytes):**
   - Find key length by Hamming distance: for candidate `KEYSIZE`, average the
     normalized Hamming distance between consecutive blocks; smallest => key length.
     (Or use Kasiski/index-of-coincidence, or `xortool`.)
   - Transpose ciphertext into `KEYSIZE` columns; solve each column as single-byte XOR.
3. **Known-plaintext / crib dragging:** if you know a substring (e.g. `flag{` or `the`),
   XOR it across every position; readable output reveals key fragment and alignment.
4. **Two-time pad (keystream reuse):** `C1 XOR C2 = P1 XOR P2`. Crib-drag a common word
   through the XOR of the two ciphertexts; a hit on one side reveals text on the other.
5. **Classical alphabetic ciphers:**
   - Caesar/ROT: brute-force all 25 shifts (ROT13 is shift 13).
   - Vigenere (letters): use key-length via IoC, then per-column frequency, or dcode.fr.
   - Substitution: frequency + quipqiup automatic solver with a crib.
   - Rail fence / columnar transposition: try rail counts 2..N, or dcode.fr.

## Commands & payloads
```python
from itertools import cycle

FREQ = b"ETAOIN SHRDLU".lower()  # rough English ranking
def score(b):
    s = 0
    for ch in b.lower():
        if ch in FREQ: s += (13 - FREQ.index(ch))
        elif ch == 0x20: s += 8
        elif 32 <= ch < 127: s += 1
        else: s -= 5
    return s

# --- single-byte XOR ---
def single_byte(ct):
    best = max(range(256), key=lambda k: score(bytes(c ^ k for c in ct)))
    return best, bytes(c ^ best for c in ct)

# --- Hamming distance & key-length guess for repeating-key XOR ---
def hamming(a, b): return sum(bin(x ^ y).count('1') for x, y in zip(a, b))
def keysize(ct, lo=2, hi=40):
    def norm(k):
        chunks = [ct[i*k:(i+1)*k] for i in range(4)]
        d = sum(hamming(chunks[i], chunks[j]) for i in range(4) for j in range(i+1,4))
        return d / (6 * k)
    return sorted(range(lo, hi+1), key=norm)[:3]

# --- repeating-key XOR: solve column by column ---
def break_repeating(ct, k):
    key = bytes(single_byte(ct[i::k])[0] for i in range(k))
    pt = bytes(c ^ kk for c, kk in zip(ct, cycle(key)))
    return key, pt

# --- crib drag over two-time pad: xored = C1 ^ C2 ---
def crib_drag(xored, crib):
    crib = crib.encode() if isinstance(crib, str) else crib
    for i in range(len(xored) - len(crib) + 1):
        out = bytes(x ^ c for x, c in zip(xored[i:i+len(crib)], crib))
        if all(32 <= o < 127 for o in out):
            print(i, out)         # readable => other plaintext at this offset

# --- Caesar / ROT brute force ---
def caesar_all(s):
    for k in range(26):
        print(k, ''.join(chr((ord(c)-97+k)%26+97) if c.islower()
              else chr((ord(c)-65+k)%26+65) if c.isupper() else c for c in s))
```
```bash
# xortool: auto key-length + probable key for repeating-key XOR
xortool -c 00 ciphertext.bin        # -c = most common plaintext byte (0x00 or 0x20)
xortool -l 8 -c 20 ciphertext.bin   # force key length 8, space as common byte
```
Interactive helpers: CyberChef "XOR Brute Force" and "Vigenere Decode";
dcode.fr for Vigenere/substitution/rail-fence; quipqiup.com for substitution solving.

## Branching logic
- Ciphertext length short, one repeating byte suspected -> single-byte XOR brute force.
- Longer, structured, key repeats -> Hamming key-length + column solve (or `xortool`).
- Two ciphertexts, same length, suspected same keystream -> crib-drag `C1^C2`.
- Only letters A-Z preserved, spacing intact -> classical: Caesar/Vigenere/substitution.
- Letters all present but reordered -> transposition (rail fence / columnar).

## Classical cipher quick reference
- **Caesar/ROT-n:** monoalphabetic shift; 25 keys, brute force. ROT13 self-inverse.
- **Atbash:** reverse-alphabet substitution (A<->Z); no key, one mapping.
- **Vigenere:** polyalphabetic; key length via IoC (~0.066 for English) or Kasiski
  repeated-trigram spacing gcd, then per-column Caesar solve.
- **Substitution:** arbitrary letter mapping; frequency analysis + quipqiup with a crib.
- **Rail fence:** zig-zag transposition; brute rail count 2..N.
- **Columnar transposition:** reorder columns by a keyword; try permutations or dcode.fr.
- **Bacon / A1Z26 / Polybius / Playfair:** recognize by structure (5-bit groups, number
  pairs, 5x5 square) and solve on dcode.fr.

## Common pitfalls
- Scoring only on letters; include space (0x20), it is the most common byte in English.
- Wrong key-length pick: check the top 2-3 candidates, not just the minimum.
- Crib dragging with too-short a crib yields false hits; use 4+ chars like `the ` / `flag`.
- Confusing repeating-key XOR key length with block size; it is purely statistical.
- Assuming ROT13 when it is an arbitrary Caesar shift, brute all 25.

## Proof / verification
Decrypted output is coherent English / contains the flag delimiter. For repeating-key,
re-encrypting plaintext with the recovered key reproduces the ciphertext exactly.

## Cross-references
- [sa--instructions--crypto--crypto-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--crypto-triage.md) (routes here), [sa--instructions--crypto--aes-block-cipher.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--aes-block-cipher.md) (CTR/OFB nonce reuse = two-time pad).

## Credible sources
- Cryptopals Set 1 (XOR, repeating-key, crib): https://cryptopals.com/sets/1
- xortool: https://github.com/hellman/xortool
- dcode.fr cipher tools: https://www.dcode.fr/en
- CTF101 XOR: https://ctf101.org/cryptography/what-is-xor/
