# Steganography Extraction Methodology

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/stego-osint-misc/steganography.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/stego-osint-misc/steganography.md)

---

---
name: steganography
category: stego-osint-misc
triggers: ["image", "png", "jpg", "jpeg", "bmp", "wav", "audio", "spectrogram", "steghide", "zsteg", "stegsolve", "outguess", "binwalk", "exiftool", "hidden data", "least significant bit", "lsb", "hidden message", "aperisolve", "appended data"]
tools: ["file", "exiftool", "strings", "binwalk", "foremost", "zsteg", "steghide", "stegseek", "outguess", "stegsolve", "aperisolve", "sonic-visualiser", "audacity", "zsteg", "pngcheck"]
priority: high
description: Decision-tree methodology for extracting hidden data from images, audio, and documents in CTF steganography challenges.
---
# Steganography Extraction Methodology

## Trigger context
Use this when a challenge provides a media file (PNG, JPG, BMP, GIF, WAV, MP3, PDF, DOCX, ZIP) and the goal is to recover a hidden flag or message. Signals: prompts like "something is hidden in this image", "look closer", "can you hear it", "the flag is in the picture", odd file sizes, or a file that renders fine but is suspiciously large.

## Objective
Recover embedded content: appended archives/files, metadata comments, LSB-encoded bytes, password-protected steghide payloads, or audio-domain messages (spectrogram text, DTMF, Morse, LSB PCM). Produce the flag or the extracted artifact and prove where it came from.

## Methodology (decision tree)
1. ALWAYS run the universal triage first (works for any file type):
   - `file` -> confirm the real type vs. the extension (renamed files are common).
   - `exiftool` -> metadata, Comment/Artist/GPS/Software fields often hold the flag directly.
   - `strings -n 6` (and `strings -e l` for UTF-16) -> plaintext, base64, URLs, `flag{`.
   - `binwalk` -> detect appended/embedded files (ZIP, PNG, gzip after EOF).
   - If binwalk finds signatures: `binwalk -e` or `foremost` to carve them out; or manually `dd` from the offset.
2. Branch on the actual file type reported by `file`:
   - PNG / BMP / GIF -> LSB and bit-plane analysis. `zsteg` (PNG/BMP only) is the highest-yield tool; `stegsolve` for visual bit-plane / color-channel inspection; `pngcheck -v` for malformed chunks / extra data after IEND.
   - JPG / JPEG -> `zsteg` does NOT work. Use `steghide` (supports JPG/BMP/WAV) and `outguess`. Both are password-capable; try empty password first, then wordlist with `stegseek`.
   - WAV / MP3 / audio -> spectrogram + LSB. Open in Sonic Visualiser or Audacity spectrogram view; check for DTMF/Morse; try `zsteg` on WAV LSB is not supported, use audio LSB scripts.
   - PDF / DOCX / XLSX / ZIP -> these are containers. `binwalk`, `unzip -l`, check for appended data past the archive's central directory, hidden layers, or embedded objects.
3. If nothing yields, escalate: try the one-stop online analyzer Aperi'Solve (runs zsteg + steghide + binwalk + exiftool + foremost + outguess + bit-plane layers automatically), or bruteforce steghide passphrases with `stegseek`.
4. When extracted data looks encoded (base64, hex, high-entropy) hand it off to the crypto/encoding methodology.

## Tool-to-format quick map
- PNG, BMP -> `zsteg`, `stegsolve`, `pngcheck`, LSB bit-plane scripts.
- JPG, JPEG -> `steghide`, `stegseek`, `outguess`, `jsteg`; NEVER `zsteg`.
- GIF -> `stegsolve` Frame Browser (data split across frames), `exiftool`, `binwalk`.
- WAV -> `steghide` (WAV supported), spectrogram (Sonic Visualiser/Audacity), LSB PCM scripts, `multimon-ng`.
- MP3 -> `mp3stego`, spectrogram, ID3 tag inspection via `exiftool`.
- PDF/DOCX/XLSX/PPTX/ZIP -> container: `unzip`, `binwalk`, `pdf-parser`, appended-data check.
- Any/unknown -> universal triage (`file`, `exiftool`, `strings`, `binwalk`).

## Commands & payloads
```bash
# --- Universal triage (run on EVERY file first) ---
file suspicious.png
exiftool suspicious.png                      # metadata, comments, GPS
strings -n 6 suspicious.png | less
strings -e l suspicious.png | grep -i flag   # UTF-16LE strings
binwalk suspicious.png                        # detect appended/embedded files
binwalk -e --dd='.*' suspicious.png           # auto-extract everything
foremost -i suspicious.png -o out_dir         # carve by file signature
xxd suspicious.png | tail                      # eyeball trailing bytes after EOF

# --- PNG / BMP: LSB & bit planes ---
zsteg suspicious.png                          # scan common LSB channel/bit combos
zsteg -a suspicious.png                        # try ALL known methods
zsteg -E 'b1,rgba,lsb,xy' suspicious.png      # extract a specific payload to stdout
pngcheck -vtp7 suspicious.png                 # malformed chunks / data after IEND
# stegsolve.jar -> GUI: cycle bit planes (Red 0..7, etc.), Data Extract, Frame Browser (GIF), Image Combiner (XOR two images)

# --- JPG (zsteg will NOT help) ---
steghide info suspicious.jpg                   # shows if a payload is embedded
steghide extract -sf suspicious.jpg           # prompts for passphrase (try empty)
steghide extract -sf suspicious.jpg -p 'secret'
stegseek suspicious.jpg /usr/share/wordlists/rockyou.txt   # fast passphrase crack -> auto-extracts
outguess -r suspicious.jpg out.txt            # OutGuess retrieval

# --- Audio ---
# Sonic Visualiser: Layer -> Add Spectrogram; hidden text/QR appears as an image
# Audacity: switch track to Spectrogram view; check Analyze for tones
# DTMF/Morse: decode tones by ear or with multimon-ng / online DTMF decoder
multimon-ng -t wav -a MORSE_CW -a DTMF suspicious.wav

# --- Documents / archives ---
unzip -l file.zip                              # list; DOCX/XLSX/PPTX are ZIPs -> unzip and grep
binwalk file.pdf; pdfdetach -list file.pdf; pdf-parser.py file.pdf

# --- One-stop online fallback ---
# Aperi'Solve (https://aperisolve.com): upload once -> runs zsteg, steghide,
# binwalk, exiftool, foremost, outguess + shows every bit-plane layer.

# --- Manual LSB extraction (when scripts/GUI unavailable) ---
python3 -c "from PIL import Image;p=Image.open('x.png').convert('RGB');\
print(''.join(str(px[c]&1) for px in p.getdata() for c in range(3))[:800])"
```

## Branching logic
- `file` says PNG/BMP -> zsteg + stegsolve. `file` says JPEG -> steghide/outguess/stegseek, skip zsteg.
- Extension/type mismatch (e.g. `.jpg` that is really a PNG) -> rename and re-triage; the mismatch itself is often the puzzle.
- Large file that renders normally -> almost always appended data; go straight to binwalk/foremost/xxd tail.
- steghide reports "could not extract any data" with empty pass -> passphrase-protected -> stegseek with rockyou.
- Image looks noisy or has faint patterns -> bit-plane (stegsolve) or LSB (zsteg -a).
- Two near-identical images provided -> XOR them (stegsolve Image Combiner) to reveal the diff.
- Audio with a broadband "buzz" -> open spectrogram before anything else.

## Common pitfalls
- Using zsteg on a JPEG or steghide on a PNG: wrong-tool-for-format is the single most common stall. zsteg = PNG/BMP; steghide = JPG/BMP/WAV; outguess = JPG.
- Forgetting `strings -e l` (UTF-16) and `exiftool` comment fields; flags frequently sit in plain metadata.
- Assuming empty steghide passphrase fails means no data; try a wordlist before giving up.
- Trusting the file extension. Always confirm with `file` / magic bytes.
- Over-carving: binwalk `-e` can produce false positives; verify carved files with `file`.
- Not checking past EOF: many challenges just concatenate a ZIP after a valid PNG (data after IEND).
- Spectrogram: too-short FFT window hides text; increase window size / zoom.

## Proof / verification
- Show the exact command and offset that produced the flag (e.g. binwalk offset + carved file hash, or `zsteg -E` payload).
- Confirm the recovered string matches the expected flag format (e.g. `flag{...}`, `picoCTF{...}`).
- For carved files, run `file` on the output to confirm it is a valid archive/image before claiming success.

## Cross-references
- Encoded output (base64/hex/rot) -> crypto & encoding methodology.
- Appended ZIP that is password-protected -> archive-cracking (fcrackzip / John zip2john).
- GPS/EXIF location data leads to a place -> see osint.md (image geolocation).
- If the "misc" file is actually a filesystem/PCAP -> forensics methodology.

## Credible sources
- CTF Support - Image Steganography: https://ctf.support/steganography/image-steganography/
- Aperi'Solve Steganography CTF Cheatsheet (decision tree): https://www.aperisolve.com/cheatsheet
- Aperi'Solve online analyzer: https://www.aperisolve.com/
- zsteg (zed-0xff): https://github.com/zed-0xff/zsteg
- steghide manual: https://steghide.sourceforge.net/documentation/manpage.php
- stegseek (steghide cracker): https://github.com/RickdeJager/stegseek
- HackTricks - Stego Tricks: https://book.hacktricks.wiki/en/crypto-and-stego/stego-tricks.html
- Sonic Visualiser: https://www.sonicvisualiser.org/
