# AES / Block Cipher Attacks

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/crypto/aes-block-cipher.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/crypto/aes-block-cipher.md)

---

---
name: aes-block-cipher
category: crypto
triggers: ["AES", "AES ECB", "ECB", "CBC", "padding oracle", "bit flipping", "IV reuse", "CTR nonce reuse", "block cipher", "MODE_ECB", "duplicate blocks", "16-byte block"]
tools: ["python", "pycryptodome", "padbuster", "CyberChef"]
priority: high
description: Block-cipher attack playbook — ECB detection and byte-at-a-time decryption, CBC bit-flipping, padding oracle, IV reuse, and CTR/stream keystream reuse.
---
# AES / Block Cipher Attacks

## Trigger context
An oracle or ciphertext using AES (or DES/3DES) in ECB, CBC, or CTR. You may control
plaintext fed to an encryption oracle, receive padding-error responses, or have two
ciphertexts under a reused IV/nonce. Block size is 16 bytes for AES (8 for DES).

## Objective
Recover plaintext or forge ciphertext by exploiting a mode-of-operation weakness,
never the AES primitive itself. The mode and what the oracle leaks decide the attack.

## Methodology (decision tree)
1. **Identify the mode.**
   - Encrypt a long run of identical bytes (e.g. 48+ `A`s). Two identical 16-byte
     ciphertext blocks appearing => **ECB** (deterministic, no diffusion).
   - IV prefix + no repeated blocks, ciphertext len % 16 == 0 => **CBC**.
   - Ciphertext length == plaintext length (not block-aligned) => **CTR/OFB/CFB stream**.
2. **ECB => byte-at-a-time decryption** (chosen-plaintext). Recover the secret suffix
   the oracle appends to your input, one byte at a time.
3. **CBC + you want to alter plaintext => bit-flipping.** Flipping byte `i` of
   ciphertext block `N` flips the same byte of decrypted block `N+1` (and garbles
   block `N`). Use to turn `admin=0` into `admin=1`.
4. **CBC + padding-valid/invalid oracle => padding oracle.** Decrypt any block without
   the key by manipulating the previous block and watching pad validity.
5. **IV reuse in CBC:** the first plaintext block is `Dec(C0) XOR IV`; a fixed/known IV
   plus known first plaintext leaks structure; reused IV across messages leaks equality
   of first blocks.
6. **CTR/OFB nonce reuse => two-time pad.** `C1 XOR C2 = P1 XOR P2`; crib-drag to recover
   both plaintexts (see xor file).

## Commands & payloads
```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad, unpad

BS = 16
def blocks(b): return [b[i:i+BS] for i in range(0, len(b), BS)]

# --- ECB detection ---
def is_ecb(ct):
    bs = blocks(ct); return len(bs) != len(set(bs))

# --- ECB byte-at-a-time (oracle appends fixed secret; assume no random prefix) ---
def ecb_decrypt(oracle):
    # find block size
    base = len(oracle(b''));  i = 1
    while len(oracle(b'A'*i)) == base: i += 1
    bs = len(oracle(b'A'*i)) - base
    known = b''
    total = len(oracle(b''))
    for _ in range(total):
        pad_len = (bs - 1 - len(known)) % bs
        prefix = b'A' * pad_len
        blk = (pad_len + len(known)) // bs      # target block index
        target = blocks(oracle(prefix))[blk]
        for g in range(256):
            guess = prefix + known + bytes([g])
            if blocks(oracle(guess))[blk] == target:
                known += bytes([g]); break
        else:
            break                                # padding byte reached -> done
    return known

# --- CBC bit-flip: make decrypted block `tgt_idx` equal `desired` ---
def cbc_bitflip(ct, tgt_idx, known_pt, desired):
    ct = bytearray(ct)
    prev = (tgt_idx - 1) * BS                    # bytes in the PREVIOUS block
    for i in range(BS):
        ct[prev + i] ^= known_pt[i] ^ desired[i]
    return bytes(ct)                             # block tgt_idx-1 is now garbage

# --- CBC padding oracle: recover one block C1 given prev block C0 ---
def padding_oracle_block(C0, C1, oracle):       # oracle(iv, block)->bool valid pad
    inter = bytearray(BS)                        # D_k(C1)
    forge = bytearray(BS)
    for pos in range(BS - 1, -1, -1):
        pad_val = BS - pos
        for k in range(pos + 1, BS):
            forge[k] = inter[k] ^ pad_val
        for g in range(256):
            forge[pos] = g
            if oracle(bytes(forge), C1):
                # guard against fake pad hit at pos==15
                inter[pos] = g ^ pad_val
                break
    return bytes(a ^ b for a, b in zip(inter, C0))   # recovered plaintext block
```
```bash
# PadBuster against a web padding oracle (cookie = base64 ciphertext, block=16)
padbuster http://target/dec.php "<b64_ciphertext>" 16 -cookies "auth=<b64_ciphertext>" -encoding 0
padbuster http://target/dec.php "<b64_ct>" 16 -encoding 0 -plaintext "admin=true"   # forge
```

## Branching logic
- Repeated ciphertext blocks -> ECB -> byte-at-a-time (add prefix-alignment if a random
  prefix precedes your input).
- Server returns distinct "padding error" vs "other error" -> padding oracle (padbuster
  or the routine above).
- You control an authenticated cookie/token in CBC and know its plaintext layout ->
  bit-flipping to escalate privilege.
- Two messages, same nonce/IV in CTR/OFB -> treat as two-time pad -> [sa--instructions--crypto--xor-and-classical.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--xor-and-classical.md).

## Common pitfalls
- ECB attack breaking when a random prefix exists: measure and pad to a block boundary
  before the standard attack.
- Padding-oracle false positive at the last byte (`\x01`): verify by also tampering the
  second-to-last byte and re-testing.
- Bit-flipping corrupts the block you edit; ensure that garbled block is disposable.
- Assuming AES is broken, it is the mode/oracle that leaks, not the cipher.
- CTR nonce reuse does NOT need block alignment; it is a stream-cipher keystream reuse.

## Proof / verification
Recovered plaintext is printable / contains the flag; forged token is accepted by the
server; for padding oracle, `unpad(recovered, 16)` succeeds and yields the message.

## Cross-references
- [sa--instructions--crypto--crypto-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--crypto-triage.md), [sa--instructions--crypto--xor-and-classical.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--xor-and-classical.md) (keystream/two-time pad), [sa--instructions--crypto--hash-cracking.md](https://raw.githubusercontent.com/BigBread57/zametki/main/08-crypto-encoding/sa--instructions--crypto--hash-cracking.md).

## Credible sources
- Cryptopals Set 2 (ECB byte-at-a-time, CBC): https://cryptopals.com/sets/2
- HackTricks padding oracle: https://book.hacktricks.wiki/en/crypto-and-stego/padding-oracle-priv.html
- CTF101 block ciphers: https://ctf101.org/cryptography/what-are-block-ciphers/
- PadBuster: https://github.com/AonCyberLabs/PadBuster
