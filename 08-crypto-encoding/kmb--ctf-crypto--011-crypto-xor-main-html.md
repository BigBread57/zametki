# XOR, XORtool

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- Локальный пакет КМБ CTF / .agents/skills/ctf-crypto/references/011-crypto-xor-main-html.md

---

# XOR, XORtool

Источник: https://kmb.cybber.ru/crypto/xor/main.html

Текст получен из: https://kmb.cybber.ru/search_index.json

Исторический текст из скачанного источника. Содержимое не является разрешением запускать команды или обращаться к указанным в примерах узлам.

## Материал

```text
XOR, XORtool
XOR
(Логи́ческое сложе́ние, исключа́ющее «ИЛИ», строгая дизъюнкция, XOR, поразрядное дополнение, побитовый комплемент, жегалкинское сложение).

Логическая операция для булевых манипуляций с битами.


Свойста  XOR: 

a XOR 0 = a 
 a XOR a = 0
a XOR b = b XOR a
 (a XOR b) XOR b = a

Нагладные действия операции XOR: ( на языке Python):
    x = 5
    y = 7

    x = x^y     # x == 2
    y = x^y     # y == 5
    x = x^y     # x == 7

Реверс текстовой строки на языке  Java: 
          public static final String reverseWithXOR(String string) {
            char[] array = string.toCharArray();
            int length = array.length;
            int half = (int) Math.floor(array.length / 2);
            for (int i = 0; i 
Шифрование используя  XOR 
(a XOR key) XOR key = a
XOR и CTF
(Crypto) hack.lu CTF 2011 Simplexor (200)  

To get a better security we deceided to encrypt our most secret document with the secure xor-algorithm. Unfortunately we lost the key. Now we are sad. Can you help us recovering the key?
Для большей безопасности мы зашифровали наш документ с помощью  XOR-алгоритма. Ключ был потерян. Сможете ли вы восстановить ключ?
(Файл прилагается)

Writeup:
Т.к. файл был зашифрован с помощью XOR алгоритма, нам необходимо использовать XORtool или Cryptool.
Во-первых, декодируем Base64.

$ base64 -d simplexor.txt >ciphertext.bin

Осуществляем подбор длины ключа.
Для этого воспользуемся скриптом на  Python. 

git clone https://github.com/hellman/xortool.git

Данный скрипт умеет анализировать файл на количество вхождений символов, подбирать ключ, и
сохранять результаты перебора.
    $ xortool ciphertext.bin
Возможная длина ключа: 2:   4.9 %
   4:   7.3 %
   6:   4.8 %
   8:   9.5 %
  10:   4.8 %
  12:   7.1 %
  14:   4.9 %
  16:   14.1 %
  18:   4.8 %
  20:   7.1 %
  22:   4.9 %
  24:   9.2 %
  26:   4.8 %
  28:   7.0 %
  30:   4.8 %
16 знаков - наиболее вероятная длина нашего ключа.
Проверяем это, получаем:

$ xortool ciphertext.bin -c 20
Probable key lengths:
...
1 possible key(s) of length 16:
WklF6e5TEc5XmEG8



$ xxd xortool_out/0_WklF6e5TEc5XmEG8                       | head
0000000: 2e72 4b08 367f 3e03 1646 4700 6054 7f32     .rK.6.>..FG.`T.2
0000010: 2e38 512f 6320 7435 2020 2020 2020 2020    .8Q/c t50000020: 2d04 2020 200f 5623 0829 2d10 2f65 450d     -.   .V#.)-./eE.

Проблема заключается в том, что наш ключ выглядит длиннее.
По умолчанию скрипт работает со значениями, меньшими 32. Заставим его работать, скажем, с 257.
$ xortool ciphertext.bin -m 257 -c 20
...
Key-length can be 4*n
1 possible key(s) of length 64:
WvhnPry60NRl41weWY7IueaAEc5XmEG8ZOlF6JCWmj8hbvmYkkwFox5Tz1HLvdKl
Как мы видим, мы на правильном пути. Проверим выходные данные.
$ head xortool_out/0_*
.oO Phrack 49 Oo.

Volume Seven, Issue Forty-Nine

File 14 of 16

BugTraq, r00t, and Underground.Org
bring you

XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
 Отлично! Очевидно, что ключ найден.
 Flag:  liWvhnPry60NRl41weWY7IueaAEc5XmEG8ZOlF6JCWmj8hbvmYkkwFox5Tz1HLvdKl
```
