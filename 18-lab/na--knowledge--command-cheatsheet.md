# Command cheatsheet

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / knowledge/command-cheatsheet.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/knowledge/command-cheatsheet.md)

---

# Command cheatsheet

Fast lookup across all categories. Full per-category detail lives in
`knowledge/tools-reference/<category>.md` — this file is the one-liner index.
Anything prefixed `.venv/` is in the project virtualenv, not on system PATH
(see `knowledge/environment.md`).

## Always-first triage (any file, including images/screenshots)

```
tools/triage_file.sh <f> <workspace-dir>   # do this one, not the four below individually —
                                             # copies into evidence/, hashes, and runs all of:
                                             #   file / exiftool / binwalk / strings
                                             # in one pass, logged to artifacts/
xxd <f> | head -50                           # only if you need to eyeball the raw header by hand
```

## Web

```
sqlmap -u "<url>" --batch --dbs
gobuster dir -u <url> -w wordlists/Discovery/Web-Content/common.txt
ffuf -u <url>/FUZZ -w wordlists/Discovery/Web-Content/raft-medium-files.txt
curl -s -i <url>                  # check headers/cookies before anything else
```

## Cryptography

```
.venv/bin/python -c "from Crypto.Util.number import *; ..."   # pycryptodome
.venv/bin/factordb <n>             # RSA modulus factorization lookup
openssl enc -d -aes-256-cbc -in ct -out pt -K <hex> -iv <hex>
echo '<b64>' | base64 -d | xxd
```

## Reverse engineering

```
r2 -A <binary>                     # auto-analyze, then `afl` to list functions, `pdf @ main`
.venv/bin/checksec --file=<binary>
objdump -d -M intel <binary> | less
gdb <binary>                        # pwndbg loads automatically
~/tools/ghidra_12.1.2_PUBLIC/support/analyzeHeadless <project-dir> <project> -import <binary> -postScript <script>
```

## Binary exploitation

```
.venv/bin/checksec --file=<binary>          # NX/PIE/RELRO/canary first, always
.venv/bin/cyclic 200                        # generate a pattern to find the overflow offset
.venv/bin/cyclic -l <4-or-8-byte-value>     # find the offset from a crashed value
.venv/bin/ROPgadget --binary <binary> --only "pop|ret"
.venv/bin/pwn template <binary> > exploit.py
```

## Forensics

```
foremost -i <image> -o out/
.venv/bin/vol -f <mem.raw> windows.info          # or linux.* for a Linux image
tshark -r <cap>.pcap -Y http                     # filtered read
tshark -r <cap>.pcap --export-objects http,out/  # pull files out of a capture
```

## Steganography

```
steghide extract -sf <img>.jpg
.venv/bin/python -c "from PIL import Image; ..."  # manual LSB inspection
binwalk -e <img>                                    # check for appended archives first
```

## OSINT

```
whois <domain>
~/tools/sherlock/.venv/bin/sherlock <username>
~/tools/theHarvester/.venv/bin/theHarvester -d <domain> -b all
.venv/bin/holehe <email>
curl -s "https://web.archive.org/web/2024*/<url>"   # wayback pivot
```

## Malware analysis

```
yara -r rules.yar <f>                        # standalone CLI
.venv/bin/capa <f>                             # capability detection
.venv/bin/floss <f>                              # deobfuscate runtime-built strings
.venv/bin/python -c "import pefile; pe = pefile.PE('<f>'); ..."
strings -n 8 <f> | grep -iE "http|\.exe|cmd|powershell"
```

## Live connections (ssh / netcat challenges)

```
nc <host> <port>                    # connect and just read first — many hand you the flag immediately
ssh <user>@<host> -p <port>
```


---

## Зависимости и ссылки вне библиотеки

Следующие файлы или ресурсы упомянуты в исходнике, но не входят в эту библиотеку документов. Их наличие и работоспособность здесь не проверялись.

- `tools/triage_file.sh`
