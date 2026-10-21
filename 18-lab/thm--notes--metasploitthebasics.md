# Metasploit: основы

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- Личные конспекты TryHackMe / notes/metasploitthebasics.md
- Личные конспекты TryHackMe / skills/thm-metasploit-basics/references/lesson.md

---

# Metasploit: основы

Источник: https://tryhackme.com/room/metasploitthebasics
Авторский конспект основных понятий Task 1–6.

Task 1–2. Framework организует возможности в модули: exploit,
auxiliary, payload, post, encoder, nop, evasion. Exploit использует
уязвимость; payload определяет действие после её использования.

Task 3. `msfconsole` запускает консоль. `help search` уточняет синтаксис;
поиск поддерживает `type:`, `platform:` и `cve:`. `info MODULE`
показывает назначение, требования и ссылки; совпадение поиска не означает
пригодность модуля для цели. `spool` сохраняет вывод консоли.

Task 4. `use MODULE` меняет контекст, `show options` показывает параметры,
`set` задаёт значение, `unset` очищает. `setg` и `unsetg` управляют
глобальными значениями. Проверяй их перед сменой цели.
RHOSTS/RPORT описывают удалённую цель; LHOST/LPORT — сторону
обратного соединения. `show payloads` перечисляет совместимые варианты.
`check` поддерживается не всеми модулями; он тоже обращается к цели.
`run`/`exploit` запускают модуль, а не просто проверяют конфигурацию.

Task 5. Различай prompt оболочки, msfconsole, выбранного модуля
и Meterpreter. `sessions` показывает сессии, `sessions -i ID`
переключает на нужную. `background` возвращает из Meterpreter
к консоли без закрытия соединения. `sessions -k ID` закрывает сессию.

Task 6. Учебный порядок: найти модуль, прочитать info, проверить контекст
и параметры, выполнить разрешённый шаг, оценить фактический результат.
В этом проекте модули не запускались и сессии не открывались.
