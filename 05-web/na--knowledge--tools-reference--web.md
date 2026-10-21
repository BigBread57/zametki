# Web exploitation — tools reference

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / knowledge/tools-reference/web.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/knowledge/tools-reference/web.md)

---

# Web exploitation — tools reference

## sqlmap
```
sqlmap -u "<url>?id=1" --batch --dbs                 # enumerate DBs
sqlmap -u "<url>?id=1" --batch -D <db> --tables
sqlmap -u "<url>?id=1" --batch -D <db> -T <t> --dump
sqlmap -r request.txt --batch                        # from a raw saved HTTP request — preferred once you have one, more reliable than rebuilding flags
sqlmap -u "<url>" --batch --data="user=a&pass=b" --level=3 --risk=2   # POST + deeper tests
```
`--batch` auto-answers prompts, essential for non-interactive use. Save a raw
request to a file (headers + body, one `curl -s -i` capture edited by hand) and
use `-r` once you know the injection point — far more reliable than
reconstructing cookies/headers as flags every time.

## gobuster / ffuf / feroxbuster (directory & vhost fuzzing)
```
gobuster dir -u <url> -w wordlists/Discovery/Web-Content/common.txt -x php,txt,bak
ffuf -u <url>/FUZZ -w wordlists/Discovery/Web-Content/raft-medium-directories.txt -mc 200,301,403
ffuf -u <url> -H "Host: FUZZ.<domain>" -w wordlists/Discovery/DNS/subdomains-top1million-5000.txt   # vhost fuzzing
~/tools/bin/feroxbuster -u <url> -w wordlists/Discovery/Web-Content/raft-medium-directories.txt      # faster on large wordlists, recurses by default
```

## nuclei (template-based vuln scanning)
```
~/tools/bin/nuclei -u <url>                      # runs its default template set — CVEs, exposed panels, misconfigs
~/tools/bin/nuclei -u <url> -tags cve,exposure     # narrow to specific template tags
```
Good as a fast first pass on a live target before manual probing, not a
replacement for reading the source/app logic in a CTF context.
`-mc` (match codes) matters — default ffuf shows everything including noisy 404s
unless you filter. `wordlists/` is the SecLists symlink — use `Discovery/Web-Content/`
for paths, `Fuzzing/` for payload lists (SQLi, XSS, LFI).

## Manual request inspection
```
curl -s -i <url>                       # headers first — look for framework fingerprints, cookies, CSP
curl -s -i -b "session=<value>" <url>  # replay with a specific cookie
curl -s -X POST <url> -d "field=value" -H "Content-Type: application/x-www-form-urlencoded"
```

## Common CTF web bug tells
- **PHP type juggling**: `==` comparisons with user input, `md5($a) == md5($b)`
  patterns, `"0e123..."` hash collisions.
- **Flask/Jinja2 SSTI**: user input reflected into a template — probe with `{{7*7}}`.
- **Node prototype pollution**: `__proto__`/`constructor.prototype` accepted in a
  JSON body that gets merged into an object.
- **Deserialization**: PHP `O:` / Java serialized-object magic bytes (`\xac\xed`)
  in a cookie or parameter.
- **JWT**: check `alg` — `none` acceptance, or HS256/RS256 key-confusion if the
  public key is discoverable.
