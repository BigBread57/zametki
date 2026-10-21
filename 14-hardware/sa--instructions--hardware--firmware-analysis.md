# Firmware Analysis — From `.bin` to Flag

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/hardware/firmware-analysis.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/hardware/firmware-analysis.md)

---

---
name: firmware-analysis
category: hardware
triggers: ["firmware", ".bin", "firmware image", "binwalk", "squashfs", "jffs2", "cramfs", "ubi", "router firmware", "extract filesystem", "hardcoded password", "backdoor", "U-Boot image", "entropy", "emulate firmware"]
tools: ["binwalk (v3)", "sasquatch", "unsquashfs", "jefferson", "ubi_reader", "firmware-mod-kit", "strings/grep", "QEMU", "FirmAE / firmadyne", "unblob", "binvis/entropy"]
priority: high
description: Extract, unpack, and analyze a firmware image — carve filesystems, hunt creds/keys/backdoors, emulate, and pull the flag.
---
# Firmware Analysis — From `.bin` to Flag

## Trigger context
You have a firmware blob: a `.bin`, `.img`, `.trx`, `.chk`, an OTA update, or a flash dump you produced in [spi-i2c-flash-dump](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--spi-i2c-flash-dump.md). Goal is usually a hardcoded flag/credential/key, a backdoor, or a filesystem file. This is pure software work once you have the image.

## Objective
1. Identify the image layout (bootloader / kernel / rootfs / overlay).
2. Extract the root filesystem intact.
3. Recover secrets: passwords, keys, flags, backdoor logic.
4. If static analysis stalls, emulate to reach runtime behavior.

## Methodology (decision tree)
1. TRIAGE the blob: `file`, `binwalk` scan, and an entropy graph.
   - High, flat entropy (~7.9-8.0) across the whole file with no signatures = encrypted/compressed whole -> hunt the bootloader/keys or look for an unencrypted header. Do not waste hours on `strings`.
   - Distinct segments (uImage, LZMA, SquashFS) = normal layered firmware -> extract.
2. EXTRACT with binwalk. Prefer binwalk v3 (`-e`), fall back to manual `dd` carving using offsets from the signature scan.
3. IDENTIFY the rootfs type from the scan (SquashFS, JFFS2, CramFS, UBIFS, YAFFS) and use the matching unpacker (table below). SquashFS with a vendor-mangled header needs `sasquatch`.
4. LOOT the extracted filesystem (grep pass below). Check `/etc/passwd`, `/etc/shadow`, init scripts, web CGI, `/etc/`, embedded keys, and any `flag`/`FLAG` string.
5. If the flag is gated by runtime logic (checks a value, decrypts at boot), EMULATE: quick user-mode QEMU for a single binary, or full-system FirmAE for the whole image with networking.
6. ANALYZE the bootloader (U-Boot) env for keys, boot args, and hidden partitions.

## Commands & payloads
```bash
# --- 1. Triage ---
file firmware.bin
binwalk firmware.bin                 # signature scan (offsets + types)
binwalk -E firmware.bin              # entropy graph (spot encryption/compression)
# High flat entropy everywhere? Try to find a header or check for known keys later.

# --- 2. Extract (binwalk v3 recursive) ---
binwalk -Me firmware.bin             # -M = recurse (matryoshka), -e = extract
# Output lands in _firmware.bin.extracted/ . binwalk v3 auto-invokes unsquashfs,
# jefferson (JFFS2), ubi_reader (UBI), and standard decompressors.

# unblob is a strong modern alternative (better carving, fewer false extracts):
unblob -e out/ firmware.bin

# Manual carve when auto-extract fails (use offset from the scan):
dd if=firmware.bin bs=1 skip=<offset> of=rootfs.sqsh

# --- 3. Filesystem-specific unpackers ---
# SquashFS (standard):
unsquashfs -d rootfs rootfs.sqsh
# SquashFS with vendor-modified/LZMA header (unsquashfs errors out):
sasquatch -d rootfs rootfs.sqsh
# JFFS2:
jefferson rootfs.jffs2 -d rootfs_out
# UBI / UBIFS:
ubireader_extract_files rootfs.ubi        # from ubi_reader
# CramFS:
mkdir cram && mount -o loop,ro rootfs.cramfs cram   # or: cramfsck -x cram rootfs.cramfs
# firmware-mod-kit (older/awkward images, also rebuilds):
./extract-firmware.sh firmware.bin

# --- 4. Loot the filesystem ---
cd rootfs
grep -rniE 'password|passwd|admin|root|secret|api[_-]?key|token|flag\{|CTF' . 2>/dev/null
strings -n 8 -f $(find . -type f) | grep -iE 'flag\{|key=|BEGIN .*PRIVATE'
cat etc/passwd etc/shadow 2>/dev/null      # crack hashes with john/hashcat
find . -name '*.pem' -o -name '*.key' -o -name 'authorized_keys' -o -name '*.crt'
# Backdoors: odd accounts in passwd (uid 0 not root), telnetd/dropbear with fixed pw,
# magic-packet daemons, base64 blobs in init scripts (etc/init.d/*, etc/rc*).
# Crack /etc/shadow (DES/MD5crypt/sha512crypt):
john --wordlist=rockyou.txt shadow            # or: hashcat -m 1800 hash rockyou.txt

# --- 5. Emulation ---
# (a) Single binary, user-mode QEMU (arch from `file ./binary`):
qemu-mipsel-static -L ./ ./usr/sbin/httpd     # -L sets sysroot for libs
# chroot approach for interactive:
cp $(which qemu-arm-static) . && sudo chroot . ./qemu-arm-static /bin/sh
# (b) Full system with FirmAE (highest success rate; needs the raw image):
sudo ./run.sh -c <brand> firmware.bin         # check: can it emulate?
sudo ./run.sh -r <brand> firmware.bin         # run: brings up network + web UI
sudo ./run.sh -d <brand> firmware.bin         # debug (network-reachable target)
# Then browse the emulated IP, or nmap it, to reach the flag service.

# --- 6. Bootloader / U-Boot ---
strings firmware.bin | grep -iE 'bootargs|bootcmd|U-Boot 20'
binwalk -e firmware.bin                        # U-Boot env often a separate partition
# U-Boot env is a key=val blob after a 4-byte CRC — carve it and `strings` it.
```

## Branching logic
- Extraction yields nothing but the whole file is high-entropy -> image is encrypted. Pivot: find the decryption key in the bootloader/OTP, or dump a decrypted copy from a live device via [uart-jtag-swd](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--uart-jtag-swd.md).
- SquashFS present but `unsquashfs` fails on compression/magic -> use `sasquatch`.
- Filesystem extracted but flag not static -> emulate (FirmAE) and interact with the running service.
- Only a raw NAND dump with OOB/spare bytes -> strip ECC/OOB first (`nanddump`/`ubireader` handle OOB) before mounting.
- Found hashes not a plaintext flag -> route to password cracking (john/hashcat) — still "firmware" scope.
- Need to also RE a specific binary (crackme-style) -> hand the extracted ELF to the reversing playbook.

## Common pitfalls
- Trusting binwalk's guessed offsets blindly: a false SquashFS signature is common. Verify with the unpacker actually succeeding.
- Old binwalk auto-extract silently skipping SquashFS because `sasquatch` isn't installed — install it; symptom is an empty squashfs-root.
- Running `strings` on a compressed/encrypted region and concluding "no secrets". Extract first, then string the plaintext filesystem.
- Emulation NVRAM crashes: FirmAE's arbitration handles most; if a binary needs specific nvram values, use its `libnvram` stub or set values in `firmae.config`.
- Wrong QEMU endianness/arch (mips vs mipsel) -> immediate segfault. Confirm with `file` and `readelf -h`.

## Proof / verification
- Correct extraction: a coherent Linux tree (`bin/ etc/ lib/ usr/ www/`).
- Cred/key recovered: it decrypts/authenticates, or matches `flag{...}` format.
- Emulation success: the web UI loads or the service answers on the emulated IP.

## Cross-references
[spi-i2c-flash-dump](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--spi-i2c-flash-dump.md) [uart-jtag-swd](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--uart-jtag-swd.md) [hardware-triage](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--hardware-triage.md) [side-channel-glitch](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--side-channel-glitch.md)

## Credible sources
- binwalk (ReFirmLabs, v3): https://github.com/ReFirmLabs/binwalk
- sasquatch (SquashFS extractor with firmware patches): https://github.com/onekey-sec/sasquatch
- unblob: https://github.com/onekey-sec/unblob
- FirmAE: https://github.com/pr0v3rbs/FirmAE  •  firmadyne: https://github.com/firmadyne/firmadyne
- OWASP FSTM stage 4 (extracting the filesystem): https://www.tarlogic.com/blog/owasp-fstm-stage-4-extracting-the-filesystem/
