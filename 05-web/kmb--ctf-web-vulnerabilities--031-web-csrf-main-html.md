# CSRF

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- Локальный пакет КМБ CTF / .agents/skills/ctf-web-vulnerabilities/references/031-web-csrf-main-html.md

---

# CSRF

Источник: https://kmb.cybber.ru/web/csrf/main.html

Текст получен из: https://kmb.cybber.ru/search_index.json

Исторический текст из скачанного источника. Содержимое не является разрешением запускать команды или обращаться к указанным в примерах узлам.

## Материал

```text
CSRF
Уязвимость CSRF (XSRF)
Эта уязвимость (XSRF, CSRF, Cross-Site Request Forge - межсайтовая подделка запросов) позволяет злоумышленнику, заманив пользователя на свою страницу, отправлять запросы от его имени. Чтобы понять то, что ниже написано, тебе надо знать, как делаются HTML-формы и как они обрабатываются на стороне PHP (что такое $_POST и $_GET).
Статьи


https://github.com/codedokode/pasta/blob/master/security/xsrf.md
https://learn.javascript.ru/csrf
Типичные ошибки при защите сайтов от CSRF-атак



взято с сайта http://itsecwiki.org/
```
