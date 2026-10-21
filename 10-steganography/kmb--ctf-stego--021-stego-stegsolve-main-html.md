# Stegsolve

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- Локальный пакет КМБ CTF / .agents/skills/ctf-stego/references/021-stego-stegsolve-main-html.md

---

# Stegsolve

Источник: https://kmb.cybber.ru/stego/stegsolve/main.html

Текст получен из: https://kmb.cybber.ru/search_index.json

Исторический текст из скачанного источника. Содержимое не является разрешением запускать команды или обращаться к указанным в примерах узлам.

## Материал

```text
Stegsolve
Описание
Stegsolve - open source решение для анализа структуры файлов,изучения устройства стереограммы, а также самостоятельного рассмотрения каждой разрядной матрицы.
Установка

Для Unix системwget http://www.caesum.com/handbook/Stegsolve.jar -O stegsolve.jarchmod +x stegsolve.jarmkdir binmv stegsolve.jar bin/  

Для Windows системСкачиваем Stegsolve с
http://www.caesum.com/handbook/Stegsolve.jarДля его работы также необходимо установить Java Runtime Environmenthttp://goo.gl/LKvNof


Примеры использования
XOR

Даны два изображения   

Открываем первое изображение, идем в Меню -> Analyse -> Image Combiner и выбираем второе изображение.  

В режиме XOR видим флаг AZADI TOWER.

Анализ данных

Дано изображение
Воспользуемся инструментом File format, находящимся во в кладке меню Analyse  
После того как сама картинка в файле кончается, за ней следует текст с флагом.

Каналы изображения

Дано изображение
При переключении каналов замечаем, что на одном из режимов присутствует часть слова "Flag"
Проверяя оставшиеся каналы, находим режим, при котором возможно прочитать запись целиком

Извлечение данных

Дано изображение
Производим анализ каналов изображения. Замечаем, что в режимах Red/Blue/Green Plane 0 исходное изображение полностью отсутствует
Меню -> Analyse -> Data ExtractВыбрав любой из приведенных выше режимов, нажимаем Preview. В окне Extract Preview видим флаг
```
