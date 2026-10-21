# Cryptography — tools reference

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / knowledge/tools-reference/cryptography.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/knowledge/tools-reference/cryptography.md)

---

# Cryptography — tools reference

## Encoding triage (do this before assuming "real" crypto)
```
echo '<text>' | base64 -d                 # base64
echo '<hex>' | xxd -r -p                  # hex
.venv/bin/python -c "print(bytes.fromhex('<hex>'))"
```
Chained encodings are common (base64-of-hex-of-base64) — decode one layer, look
at the result, decide if it's plaintext, another encoding, or binary before
assuming you're stuck.

## RSA
```
~/tools/RsaCtfTool/.venv/bin/RsaCtfTool --publickey key.pub --uncipher <ct>   # tries ~70 known attacks automatically — run this before hand-rolling one
.venv/bin/factordb <n>                                   # check FactorDB first — many CTF moduli are already factored publicly
.venv/bin/python -c "
from Crypto.Util.number import inverse
p, q, e = ..., ..., 65537
n = p*q; phi = (p-1)*(q-1)
d = inverse(e, phi)
"
```
Conditions to check for, roughly in order of how often they show up:
- **Small `e`** (e=3) with no padding → cube-root attack.
- **Shared modulus** (two ciphertexts, same `n`, different `e`) → common-modulus attack.
- **Small private exponent `d`** relative to `n` → Wiener's attack.
- **`p`/`q` close together** → Fermat factorization.
- **Reused/related nonces** in signature schemes → key recovery from two signatures.

## XOR
```
.venv/bin/python -c "
ct = bytes.fromhex('<hex>')
for k in range(256):
    print(bytes(b ^ k for b in ct))
"     # single-byte key brute force
```
For multi-byte repeating-key XOR: guess key length via Hamming-distance /
index-of-coincidence, then solve each column as single-byte XOR — write this by
hand, it's short, don't reach for a tool.

## Block ciphers
- **ECB mode tell**: repeated 16-byte blocks in the ciphertext (`.venv/bin/python`
  slice-and-compare) — if you see repetition, it's ECB and likely exploitable via
  byte-at-a-time decryption if you can influence the plaintext.
- **CBC padding oracle**: if the service distinguishes padding errors from other
  errors, that's the whole vulnerability — script the oracle with `requests`.

## Hashes
```
hashcat -m <mode> -a 0 hash.txt wordlists/Passwords/rockyou.txt
john --wordlist=wordlists/Passwords/rockyou.txt hash.txt
```
`-m` mode number matters — `hashcat --help | grep -i <algo>` to find it rather
than guessing.

## JWT
Decode header/payload without a library (they're just base64url): split on `.`,
base64url-decode the first two segments. Check `alg` for `none`, check if HS256
signing accepts the RS256 public key as the HMAC secret (key confusion).
