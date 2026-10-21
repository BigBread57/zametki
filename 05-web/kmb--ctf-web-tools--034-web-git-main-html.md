# .git

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- Локальный пакет КМБ CTF / .agents/skills/ctf-web-tools/references/034-web-git-main-html.md

---

# .git

Источник: https://kmb.cybber.ru/web/git/main.html

Текст получен из: https://kmb.cybber.ru/search_index.json

Исторический текст из скачанного источника. Содержимое не является разрешением запускать команды или обращаться к указанным в примерах узлам.

## Материал

```text
.git
Чаще всего если вы в ctf таске (или в боевых условиях) встречаете папку .git, то обычно на нее права 403 и запрещенный листинг.Опишу какие файлы находятся в этой папке
|-- COMMIT_EDITMSG
|-- FETCH_HEAD
|-- HEAD
|-- ORIG_HEAD
|-- branches
|-- config
|-- description
|-- hooks
|   |-- applypatch-msg
|   |-- commit-msg
|   |-- post-commit
|   |-- post-receive
|   |-- post-update
|   |-- pre-applypatch
|   |-- pre-commit
|   |-- pre-rebase
|   |-- prepare-commit-msg
|   `-- update
|-- index
|-- info
|   `-- exclude
|-- logs
|   |-- HEAD
|   `-- refs
|-- objects
`-- refs
    |-- heads
    |-- remotes
    |-- stash
    `-- tags
Описание файлов
Мы рассмотрим самые важные файлы.
Некоторая информация была взята со статьи http://habrahabr.ru/post/143079/
index
В этом файле находится перечисление всех файлов данной директории.Пример:
config
Как вы уже поняли в нем находятся конфигурации данного репозитория.Пример:
[core]
    repositoryformatversion = 0
    filemode = true
    bare = false
    logallrefupdates = true
description
Файл описания данного репозитория.Пример(файл пуст):
    Unnamed repository; edit this file 'description' to name the repository.
HEAD
Файл со ссылкой на текущую ветку (чаще всего ref: refs/heads/master)Пример(файл по умолчанию):
    ref: refs/heads/master
COMMIT_EDITMSG
Файл с последним комментарием последнего редактора (опять же не часто используется, но в некоторых ctf может помочь)Пример(файл по умолчанию):
# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
# Explicit paths specified without -i or -o; assuming --only paths...
#
# Committer: is_work 
#
# On branch master
#
# Initial commit
#
# Changes to be committed:
#    new file:   secret_database_dump.txt
#
logs/HEAD
Файл с историей редактирования кодаПример:
0000000000000000000000000000000000000000 3c4992205aba2077cbf87fc7cde900fabecd1140 root  1412673432 +0000commit (initial): Hurr durr
3c4992205aba2077cbf87fc7cde900fabecd1140 ec972f9af79a09129021a30e7f08099aa2b8a81d John Doe  1412673456 +0000   commit (amend): Hurr durr
ec972f9af79a09129021a30e7f08099aa2b8a81d 0b4d6fe0adf809c4e7b7a0d47132600b68f79fda root  1417230572 +0000commit: My precious flag now exists
Утилиты
Утилита, помогающая в данной ситуации https://github.com/kost/dvcs-ripperТакже, утилита для парсинга .git/index : https://github.com/sbp/gin

взято с сайта http://itsecwiki.org/
```
