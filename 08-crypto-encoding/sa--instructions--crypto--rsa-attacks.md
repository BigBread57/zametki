# RSA Attacks

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/crypto/rsa-attacks.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/crypto/rsa-attacks.md)

---

---
name: rsa-attacks
category: crypto
triggers: ["RSA", "n e c", "public key", "small e", "e=3", "cube root", "Wiener", "low d", "common modulus", "Hastad", "broadcast", "Fermat", "close primes", "factordb", "RsaCtfTool", "PEM"]
tools: ["RsaCtfTool", "python", "sympy", "pycryptodome", "sage", "factordb", "gmpy2"]
priority: high
description: Decision tree for RSA CTF challenges given n,e,c or a public key, from factordb lookup through low-exponent, Wiener, common-modulus, and Hastad attacks.
---
# RSA Attacks

## Trigger context
Challenge gives some subset of `n` (modulus), `e` (public exponent), `c` (ciphertext),
possibly a `.pem` public key, sometimes multiple keys/ciphertexts, or a partial private
key. Goal: recover `m` (the flag) and/or `d`.

## Objective
Recover plaintext `m` where `c = m^e mod n`. The winning move depends entirely on
which parameter is weak: tiny `e`, tiny `d`, close primes, shared `n`, or an `n`
already in factordb.

## Methodology (decision tree)
1. **Extract params.** From PEM: `openssl rsa -pubin -in key.pem -text -noout` or the
   Python snippet below. Note bit-length of `n`, value of `e`.
2. **factordb first, always.** Paste `n` into factordb; if it returns `p`, `q`, you win
   instantly regardless of e. Small/CTF-reused primes live here.
3. **Small e (e = 3, 5, 17):**
   - If `m^e < n` (message short, no padding) -> plaintext is just the integer `e`-th
     root of `c`. Take exact root with `gmpy2.iroot(c, e)`.
   - Else if `c` broadcast under same small `e` to `e` different moduli -> **Hastad
     broadcast**: CRT-combine the ciphertexts, take `e`-th root of the result.
4. **Fermat factorization** when `p` and `q` are close (|p-q| small): iterate
   `a = ceil(sqrt(n))` upward until `a^2 - n` is a perfect square.
5. **Wiener's attack** when `d` is small (`d < n^0.25`) and `e` is large (close to `n`):
   continued-fraction convergents of `e/n` reveal `d`.
6. **Common modulus** when the same `n` and same `m` are encrypted under two coprime
   exponents `e1, e2`: use extended-Euclid `a*e1 + b*e2 = 1`, then
   `m = c1^a * c2^b mod n`.
7. **Partial key / smooth primes / ROCA / Boneh-Durfee:** hand to RsaCtfTool `--attack all`.
8. **Manual decrypt** once factored: `phi = (p-1)(q-1)`, `d = inverse(e, phi)`,
   `m = pow(c, d, n)`.

## Commands & payloads
```python
from Crypto.Util.number import long_to_bytes, inverse, GCD
from Crypto.PublicKey import RSA
import gmpy2, sympy

# --- parse a PEM public key ---
k = RSA.import_key(open('key.pem').read())
n, e = k.n, k.e; print(n.bit_length(), e)

# --- e-th root when m^e < n (unpadded small e) ---
r, exact = gmpy2.iroot(c, e)
if exact: print(long_to_bytes(int(r)))

# --- Fermat (close primes) ---
def fermat(n):
    a = gmpy2.isqrt(n) + 1
    while not gmpy2.is_square(a*a - n): a += 1
    b = gmpy2.isqrt(a*a - n); return int(a-b), int(a+b)
p, q = fermat(n)

# --- Common modulus (same n, e1,e2 coprime) ---
def common_modulus(c1, c2, e1, e2, n):
    g, a, b = sympy.gcdex(e1, e2)  # a*e1 + b*e2 = 1
    a, b = int(a), int(b)
    if a < 0: c1 = inverse(c1, n); a = -a
    if b < 0: c2 = inverse(c2, n); b = -b
    return (pow(c1, a, n) * pow(c2, b, n)) % n

# --- Hastad broadcast (same e, different n_i) ---
def hastad(cs, ns, e):
    from sympy.ntheory.modular import crt
    C, _ = crt(ns, cs)
    r, exact = gmpy2.iroot(int(C), e)
    return long_to_bytes(int(r)) if exact else None

# --- generic manual decrypt after factoring ---
phi = (p-1)*(q-1); d = inverse(e, phi); m = pow(c, d, n)
print(long_to_bytes(m))
```
```bash
# RsaCtfTool: throw everything at a public key
RsaCtfTool --publickey key.pem --uncipherfile cipher.bin --attack all
# from raw params (no PEM) — let it try every factorization/attack
RsaCtfTool -n <N> -e <E> --uncipher <C> --attack all --private
# target one attack when you already know the weakness
RsaCtfTool --publickey key.pem --attack wiener --private
RsaCtfTool --publickey key.pem --attack fermat --private
```

## Branching logic
- `e == 3` and short flag -> cube root (`iroot`). No luck -> suspect Hastad if multiple ns.
- `e` huge (~n) -> Wiener / Boneh-Durfee (small d).
- Two ciphertexts, same n, different e -> common modulus.
- Same message + same e across many recipients -> Hastad broadcast.
- factordb returns factors -> skip attacks, go straight to manual decrypt.
- Nothing works -> `RsaCtfTool --attack all`; inspect n for smoothness (Pollard p-1),
  ROCA fingerprint, or a leaked bit of p (`--partial`).

## Attack-to-weakness quick reference
| Symptom / parameter | Attack | Tool/route |
|---------------------|--------|-----------|
| `n` in factordb | direct factor lookup | factordb, then manual decrypt |
| `e` small (3/5), short m | `e`-th root | `gmpy2.iroot` |
| same `m`, small `e`, `e` moduli | Hastad broadcast | CRT + root |
| `\|p-q\|` small | Fermat | `fermat(n)` / RsaCtfTool |
| `d` small (`d<n^0.25`), `e~n` | Wiener | RsaCtfTool `--attack wiener` |
| `d<n^0.292` | Boneh-Durfee | RsaCtfTool (lattice) |
| same `n`, coprime `e1,e2` | common modulus | extended Euclid |
| `n` smooth (p-1 smooth) | Pollard p-1 | RsaCtfTool |
| known high/low bits of `p`/`d` | partial key | RsaCtfTool `--partial` |
| 2048-bit key, odd prime gen | ROCA | RsaCtfTool `--attack roca` |

## Common pitfalls
- Forgetting padding: OAEP/PKCS#1v1.5 breaks the naive `iroot`; strip/handle padding.
- Assuming `iroot` is exact, always check the `exact` flag or `r**e == c`.
- Common-modulus needs `gcd(e1,e2)==1`; negative exponents need modular inverse.
- Wiener only works for genuinely small d; large-ish d needs Boneh-Durfee (lattice).
- Mixing up `long_to_bytes` byte order or leaving leading null bytes off the flag.

## Proof / verification
`long_to_bytes(m)` decodes to printable text containing the flag delimiter. Sanity
check factoring with `p * q == n` and `p, q` both prime (`sympy.isprime`). For a full
key, verify `pow(pow(m, e, n), d, n) == m`.

## Cross-references
- [sa--instructions--crypto--crypto-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--crypto-triage.md) (identify n,e,c), [sa--instructions--crypto--xor-and-classical.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--xor-and-classical.md) (if plaintext still encoded),
  [sa--instructions--crypto--hash-cracking.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--hash-cracking.md).

## Credible sources
- RsaCtfTool (attack list & usage): https://github.com/RsaCtfTool/RsaCtfTool
- FactorDB: http://factordb.com/
- CTF101 RSA: https://ctf101.org/cryptography/what-is-rsa/
- HackTricks RSA attacks: https://book.hacktricks.wiki/en/crypto-and-stego/rsa.html
