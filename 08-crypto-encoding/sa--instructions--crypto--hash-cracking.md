# Hash Identification & Cracking

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/crypto/hash-cracking.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/crypto/hash-cracking.md)

---

---
name: hash-cracking
category: crypto
triggers: ["hash", "hashcat", "john", "md5", "sha1", "sha256", "bcrypt", "ntlm", "crack password", "wordlist", "rockyou", "rules", "length extension", "hashpump", "identify hash"]
tools: ["hashcat", "john", "hashid", "name-that-hash", "hashpump", "hashpumpy", "python"]
priority: high
description: Identify an unknown hash, pick the right hashcat/john mode, run wordlist+rule attacks, and perform hash length-extension forgery with hashpump.
---
# Hash Identification & Cracking

## Trigger context
A fixed-width hex digest, a `$id$salt$hash` crypt string, an NTLM dump, or a signed
`hash(secret || data)` MAC where you control appended data. Goal: recover the preimage
(password/flag) or forge a valid extended MAC.

## Objective
Either crack the hash to its plaintext, or, for a secret-prefix MAC, forge
`hash(secret || data || padding || attacker_data)` without knowing the secret.

## Methodology (decision tree)
1. **Identify the hash** by length + structure, then confirm with a tool:
   - 32 hex = MD5 or NTLM; 40 hex = SHA-1; 64 hex = SHA-256; 128 hex = SHA-512.
   - `$1$` = md5crypt, `$2a$/$2b$/$2y$` = bcrypt, `$5$` = sha256crypt, `$6$` = sha512crypt,
     `$apr1$` = Apache MD5, `$krb5tgs$` = Kerberoast, `$y$` = yescrypt.
   - Run `hashid`, `name-that-hash` (`nth`), or `hashcat --identify` for the mode number.
2. **Pick the mode** (hashcat `-m`): 0=MD5, 100=SHA1, 1400=SHA256, 1700=SHA512,
   1000=NTLM, 3200=bcrypt, 1800=sha512crypt, 500=md5crypt, 5600=NetNTLMv2,
   13100=Kerberoast, 22000=WPA. In John use the `--format=` name (e.g. `raw-md5`,
   `bcrypt`, `Raw-SHA256`).
3. **Attack ladder (fast to slow):** straight wordlist -> wordlist + rules -> combinator
   -> mask/brute force. Start with `rockyou.txt`.
4. **Salted/slow hashes (bcrypt, sha512crypt):** wordlist+rules only; brute force is
   infeasible. Keep candidate lists tight and rule-driven.
5. **Length extension** when a scheme uses `H(secret || message)` as a MAC (MD5/SHA1/
   SHA2 family, Merkle-Damgard). Given the digest and `len(secret)`, append data and
   forge a new valid digest with hashpump, without the secret.

## Commands & payloads
```bash
# --- identify ---
hashid '5f4dcc3b5aa765d61d8327deb882cf99'
nth -t '$2b$12$...'                         # name-that-hash
hashcat --identify hashes.txt

# --- crack: straight wordlist ---
hashcat -m 0 -a 0 hashes.txt rockyou.txt
# wordlist + best64 rules (highest ROI)
hashcat -m 0 -a 0 hashes.txt rockyou.txt -r rules/best64.rule
# mask / brute force: 8-char lower+digit
hashcat -m 1400 -a 3 hashes.txt '?l?l?l?l?l?l?d?d'
# show cracked results
hashcat -m 0 hashes.txt --show

# --- John equivalents ---
john --format=raw-md5 --wordlist=rockyou.txt hashes.txt
john --format=raw-md5 --wordlist=rockyou.txt --rules=Jumbo hashes.txt
john --show hashes.txt
# unshadow before cracking Linux shadow
unshadow /etc/passwd /etc/shadow > crack.txt
```
```python
# --- hash length extension with hashpumpy ---
# Known: original MAC = H(secret || known_data), and len(secret).
import hashpumpy
new_sig, new_msg = hashpumpy.hashpump(
    orig_sig='...hexdigest...',    # digest of secret||known_data
    orig_data='user=guest',        # the known message (without secret)
    data_to_add='&admin=true',     # attacker-controlled suffix
    key_length=14)                 # length of the unknown secret in BYTES
print(new_sig)                     # forged valid digest
print(new_msg)                     # bytes incl. glue padding to submit as message
# If key_length unknown, loop it: for kl in range(1, 65): try that value.
```
```bash
# hashpump CLI equivalent
hashpump -s <orig_sig> -d 'user=guest' -a '&admin=true' -k 14
```

## Mode quick reference (hashcat -m)
| Hash | -m | Hash | -m |
|------|----|------|----|
| MD5 | 0 | sha512crypt `$6$` | 1800 |
| SHA1 | 100 | md5crypt `$1$` | 500 |
| SHA256 | 1400 | NTLM | 1000 |
| SHA512 | 1700 | NetNTLMv2 | 5600 |
| bcrypt `$2*$` | 3200 | Kerberoast `$krb5tgs$` | 13100 |
| sha256crypt `$5$` | 7400 | WPA-PBKDF2/PMKID | 22000 |

## Branching logic
- Unsalted fast hash (MD5/SHA1/SHA256) -> wordlist -> rules -> mask.
- bcrypt/sha512crypt (slow, salted) -> wordlist + targeted rules only; do NOT mask-brute.
- NTLM/Net-NTLMv2/Kerberoast -> the specific mode (1000/5600/13100) + rules.
- MAC is `H(secret || data)` and you can append -> length extension (hashpump), not cracking.
- HMAC (`H(k, m)` proper) -> length extension does NOT apply; must crack/leak the key.

## Common pitfalls
- Wrong hashcat mode for raw vs salted variants (e.g. 0 vs 10 for `md5($pass.$salt)`).
- Confusing MD5 (0) with NTLM (1000): both 32 hex; context/tool disambiguates.
- Length extension only works on Merkle-Damgard (MD5/SHA1/SHA256/SHA512), NOT SHA-3,
  BLAKE2, or true HMAC.
- Guessing `key_length` wrong for hashpump: iterate all plausible lengths.
- Forgetting the forged message includes raw glue-padding bytes; submit them verbatim
  (URL-encode if sent over HTTP).

## Wordlists & rules reference
- Primary wordlist: `rockyou.txt` (SecLists/`/usr/share/wordlists/rockyou.txt`). For
  targeted challenges also try `SecLists/Passwords/` and CTF-specific leaks.
- High-ROI rules (hashcat `rules/`): `best64.rule` (start here), `d3ad0ne.rule`,
  `dive.rule`, `OneRuleToRuleThemAll.rule` (broad but slow). John: `--rules=Jumbo`.
- Mask attack charsets: `?l` lower, `?u` upper, `?d` digit, `?s` symbol, `?a` all,
  `?b` raw byte. Custom sets via `-1 ?l?d` then reference `?1`.
- Combinator (`-a 1`) joins two wordlists; hybrid (`-a 6`/`-a 7`) appends/prepends a mask
  to wordlist words — good for `password2024!`-style patterns.
- Estimate feasibility with `--keyspace`; watch `Speed`/`ETA` in the status line and
  abort brute force early on slow salted hashes.

## Proof / verification
Cracked: `hashcat --show` / `john --show` prints `hash:plaintext`, and re-hashing the
plaintext reproduces the digest. Length extension: the server accepts `new_msg` with
`new_sig`, or locally `H(known_secret + new_msg) == new_sig`.

## Cross-references
- [sa--instructions--crypto--crypto-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--crypto-triage.md) (recognize a digest), [sa--instructions--crypto--aes-block-cipher.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--aes-block-cipher.md) (MAC/auth context).

## Credible sources
- Hashcat example hashes (modes): https://hashcat.net/wiki/doku.php?id=example_hashes
- John the Ripper formats: https://www.openwall.com/john/doc/
- HashPump: https://github.com/bwall/HashPump
- HackTricks length extension: https://book.hacktricks.wiki/en/crypto-and-stego/hash-length-extension-attack.html
