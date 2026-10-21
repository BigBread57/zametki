# Web Reconnaissance & Content Discovery

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/web/web-recon.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/web/web-recon.md)

---

---
name: web-recon
category: web
triggers: [recon, enumeration, "content discovery", "directory brute force", ffuf, gobuster, dirsearch, feroxbuster, "vhost fuzzing", "subdomain enumeration", "parameter discovery", arjun, "tech fingerprint", whatweb, "404 page", "hidden endpoint"]
tools: [ffuf, gobuster, dirsearch, feroxbuster, arjun, whatweb, httpx, gau, waybackurls]
priority: high
description: First-move web enumeration: fingerprint stack, brute force content, fuzz vhosts/subdomains, discover hidden params.
---
# Web Reconnaissance & Content Discovery

## Trigger context
- A web service is in scope (port 80/443/8080/8000/8443) and you have no map of it.
- Landing page is sparse, a login form, a default framework page, or a plain 200/403.
- You need hidden dirs, backup files, API routes, admin panels, virtual hosts, or unlinked GET/POST params.
- Signals: custom 404 length, `Server:`/`X-Powered-By` headers, framework favicons, wildcard DNS, `robots.txt` disallows.

## Objective
Build a complete attack surface map: technology stack, all reachable paths/files, virtual hosts, subdomains, and every hidden parameter — feeding specific vuln-class files next.

## Methodology (decision tree)
1. Fingerprint the stack FIRST (whatweb, headers, favicon hash, error pages). Stack drives every later choice (e.g. PHP -> LFI/RFI, Java -> deserialization/SSTI Freemarker).
   1a. TECH & VERSION fingerprinting: identify product + exact version (meta generator, `X-Powered-By`, `?ver=`, favicon hash, `nuclei -t http/technologies`). If a known CMS/app (WordPress/Drupal/Joomla/Tomcat/Jenkins/phpMyAdmin) or any leaked version string appears -> go straight to [cms-and-known-exploits](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--cms-and-known-exploits.md) to map it to public CVEs/PoCs before manual bug-hunting.
2. Grab `robots.txt`, `sitemap.xml`, `/.git/`, `/.well-known/`, JS source files. Parse JS for endpoints and API keys.
3. Calibrate a baseline: record default 404 status + size so you can filter (`-fc`/`-fs`/`-ac`).
4. Directory/file brute force with a good wordlist; add relevant extensions for the detected stack.
5. Recurse into discovered dirs; re-run with backup/config extensions on interesting hits.
6. If the app uses `Host`-based routing or a CTF hints at a hostname, fuzz vhosts. Add discovered vhost to `/etc/hosts`.
7. For real domains, enumerate subdomains (passive + brute). CTF: usually vhost fuzzing instead.
8. On each dynamic endpoint, run parameter discovery (arjun / ffuf param fuzz) before testing injection classes.
9. Mine historical URLs (gau/waybackurls) for dead params and old endpoints.

## Commands & payloads
```bash
# 1. Fingerprint
whatweb -a3 http://TARGET/
curl -sI http://TARGET/ | grep -iE 'server|x-powered|x-generator|set-cookie|location'
# favicon hash -> shodan/framework id
curl -s http://TARGET/favicon.ico | md5sum
httpx -u http://TARGET -title -tech-detect -status-code -server
# 1a. version + known-CVE sweep (feeds cms-and-known-exploits)
curl -s http://TARGET/ | grep -iE '<meta name="generator"|\?ver=[0-9.]'
nuclei -u http://TARGET -t http/technologies/ -t http/cves/

# 2. Quick wins
for p in robots.txt sitemap.xml .git/HEAD .env .well-known/security.txt; do
  echo "== $p =="; curl -s -o /dev/null -w "%{http_code} %{size_download}\n" http://TARGET/$p; done

# 4. Directory discovery (ffuf) — -ac auto-calibrates the 404 baseline
ffuf -u http://TARGET/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -ac -recursion -recursion-depth 2 -e .php,.txt,.bak,.zip,.old,.json -mc all -fc 404 -t 60 -o ffuf_dirs.json

# gobuster equivalent
gobuster dir -u http://TARGET/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -x php,txt,html,bak -t 50 -b 404,403 -o gobuster_dirs.txt

# dirsearch (auto-extensions, recursion)
dirsearch -u http://TARGET/ -e php,asp,aspx,jsp,html,zip,bak -r -R 2 --random-agent

# feroxbuster (fast recursive)
feroxbuster -u http://TARGET/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt \
  -x php,bak,zip -d 3 -C 404

# 6. Virtual host fuzzing — filter by response SIZE, not status (all vhosts return 200)
ffuf -u http://TARGET/ -H "Host: FUZZ.target.htb" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -fs BASELINE_SIZE -t 60
# add hit to hosts:  echo "TARGET admin.target.htb" | sudo tee -a /etc/hosts

# 7. Subdomains (real domains)
subfinder -d target.com -all -silent | httpx -silent -title -sc
ffuf -u http://target.com/ -H "Host: FUZZ.target.com" -w subdomains.txt -ac

# 8. Parameter discovery
arjun -u http://TARGET/page.php -m GET,POST -t 20 -oT arjun.txt
ffuf -u "http://TARGET/page.php?FUZZ=test" -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs BASELINE

# 9. Historical URLs
gau target.com | tee gau.txt ; waybackurls target.com | tee wb.txt
cat gau.txt | grep -oP '\?.*' | sort -u   # extract param names
```

## Branching logic
- If a known CMS/app or leaked version string is fingerprinted -> [cms-and-known-exploits](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--cms-and-known-exploits.md) FIRST (public CVE often beats manual hunting).
- If dynamic params (`?id=`, `?file=`, `?url=`) found -> test [sql-injection](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--sql-injection.md), [command-injection](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--command-injection.md), [ssti](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--ssti.md), and SSRF/LFI.
- If reflected input in HTML/JS response -> [xss](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--xss.md).
- If template-rendered output or profile/preview fields -> [ssti](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--ssti.md).
- If `/.git/` returns 200 -> dump with git-dumper, review source for creds/logic.
- If a login/admin panel found -> default creds, then auth-based injection tests.
- If upload functionality -> test file upload -> webshell (pairs with [command-injection](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--command-injection.md)).
- If all dirs 403 but content differs -> try `-mc all`, header tricks (`X-Forwarded-For`, `X-Original-URL`).

## Common pitfalls
- Not calibrating the 404 baseline -> flood of false positives. Always `-ac`/`-fc`/`-fs`.
- Filtering vhosts by status code (they all 200) instead of size/words -> misses everything. Use `-fs`.
- Wrong wordlist for the stack (php exts on a Node app). Fingerprint first.
- Forgetting to add discovered vhosts to `/etc/hosts` -> connection refused.
- Rate limits / WAF: lower `-t`, add `-p 0.1` delay, `--random-agent`.
- HTTPS with self-signed cert: add `-k` (curl) / ffuf ignores by default.

## Proof / verification
- Confirm a discovered path is real: `curl -s http://TARGET/PATH | head` and compare size vs 404 baseline.
- Screenshot/record status+size for each finding. Flags often sit in `robots.txt`, backup files (`.bak`, `~`, `.swp`), `/flag`, `/.git/`, or source comments.
- `grep -rIED 'flag\{|FLAG\{|CTF\{' downloaded_source/` after dumping any source.

## Cross-references
[cms-and-known-exploits](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--cms-and-known-exploits.md) · [sql-injection](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--sql-injection.md) · [command-injection](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--command-injection.md) · [ssti](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--ssti.md) · [xss](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--xss.md)

## Credible sources
- https://www.thehacker.recipes/web/recon/virtual-host-fuzzing
- https://github.com/ffuf/ffuf/wiki
- https://github.com/s0md3v/Arjun
- https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/01-Information_Gathering/
- https://book.hacktricks.wiki/en/network-services-pentesting/pentesting-web/index.html
