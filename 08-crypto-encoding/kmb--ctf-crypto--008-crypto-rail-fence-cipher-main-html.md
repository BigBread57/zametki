# Взлом Rail fence cipher

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- Локальный пакет КМБ CTF / .agents/skills/ctf-crypto/references/008-crypto-rail-fence-cipher-main-html.md

---

# Взлом Rail fence cipher

Источник: https://kmb.cybber.ru/crypto/rail_fence_cipher/main.html

Текст получен из: https://kmb.cybber.ru/search_index.json

Исторический текст из скачанного источника. Содержимое не является разрешением запускать команды или обращаться к указанным в примерах узлам.

## Материал

```text
Взлом Rail fence cipher
Шифр Rail fence относится к перестановочным шифрам и взламывается методом перебора. Для реализации можно использовать инструмент RailFencePython, модифицировав тестовый файл testRailFence.py
Пример
Имеется зашифрованная строка:  

AaY--rpyfneJBeaaX0n-,ZZcs-uXeeSVJ-sh2tioaZ}slrg,-ciE-anfGt.-eCIyss-TzprttFliora{GcouhQIadctm0ltt-FYluuezTyorZ-


    from railFence import decryptRailFence  
    for i in xrange(1,255):  
        print decryptRailFence("AaY--rpyfneJBeaaX0n-,ZZcs-uXeeSVJ-sh2tioaZ}slrg,-ciE-anfGt.-eCIyss-TzprttFliora{GcouhQIadctm0ltt-FYluuezTyorZ-", i, 0);

После запуска, просматривая вывод, можно найти строку:

 A-fence-is-a-structure-that-encloses-an-area,-SharifCTF{QmFzZTY0IGlzIGEgZ2VuZXJpYyB0ZXJt},-typically-outdoors.


взято с сайта http://itsecwiki.org/
```
