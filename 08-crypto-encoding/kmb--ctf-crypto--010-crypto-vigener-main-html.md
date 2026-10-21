# Шифр Виженера

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- Локальный пакет КМБ CTF / .agents/skills/ctf-crypto/references/010-crypto-vigener-main-html.md

---

# Шифр Виженера

Источник: https://kmb.cybber.ru/crypto/vigener/main.html

Текст получен из: https://kmb.cybber.ru/search_index.json

Исторический текст из скачанного источника. Содержимое не является разрешением запускать команды или обращаться к указанным в примерах узлам.

## Материал

```text
Шифр Виженера
 
Шифр Виженера — метод полиалфавитного шифрования буквенного текста с использованием ключевого слова.
Описание алгоритма

В шифре Цезаря каждая буква алфавита сдвигается на несколько позиций; например в шифре Цезаря при сдвиге +3, A стало бы D, B стало бы E и так далее. Шифр Виженера состоит из последовательности нескольких шифров Цезаря с различными значениями сдвига. Для зашифровывания может использоваться таблица алфавитов, называемая tabula recta или квадрат (таблица) Виженера. Применительно к латинскому алфавиту таблица Виженера составляется из строк по 26 символов, причём каждая следующая строка сдвигается на несколько позиций. Таким образом, в таблице получается 26 различных шифров Цезаря. На каждом этапе шифрования используются различные алфавиты, выбираемые в зависимости от символа ключевого слова.
 
Квадрат Виженера
 Например, предположим, что исходный текст имеет вид:
ATTACKATDAWN
Человек, посылающий сообщение, записывает ключевое слово («LEMON») циклически до тех пор, пока его длина не будет соответствовать длине исходного текста:
LEMONLEMONLE
Первый символ исходного текста A зашифрован последовательностью L, которая является первым символом ключа. Первый символ L шифрованного текста находится на пересечении строки L и столбца A в таблице Виженера. Точно так же для второго символа исходного текста используется второй символ ключа; то есть второй символ шифрованного текста X получается на пересечении строки E и столбца T. Остальная часть исходного текста шифруется подобным способом.
Исходный текст:   ATTACKATDAWN

Ключ: LEMONLEMONLE

Зашифрованный текст: LXFOPVEFRNHR
Расшифрование

 Криптоанализ шифра может быть построен в два этапа:

Поиск длины ключа. Можно анализировать распределение частот в зашифрованном тексте с различным прореживанием. То есть брать текст, включающий каждую 2-ю букву зашифрованного текста, потом каждую 3-ю и т. д. Как только распределение частот букв будет сильно отличаться от равномерного (например, по энтропии), то можно говорить о найденной длине ключа.
Криптоанализ. Совокупность l шифров Цезаря (где l — найденная длина ключа), которые по отдельности легко взламываются.

Практический пример

Задание:
Необходимо расшифровать данный текст:

LoatuvftYejeerzAgibeejwzriyazfrkknxefvoxvhanvmsxlizyjzhnxmvhnjwyhnonafjgmiunfrbjxnzrrgfkgearfywv.Bnotfrqgwesiprqzbvotvvgomcumozbklszuqzsypizhslbjtmkngrzggdgpccwkwsiireqk,tsceycoyvuztveu-kwgktrtvthlugvvgggdonafjgmibengdxhaihrj.HnxUtiivfybte’scfgomiunvehnxngtvfbgeutiivfybterneyoggypefjoweyprigatsovrvjowetcrkcomsgcuzsbxmkngj,ovhsotvmsofamenergiaysvfblhrkxpvzrxnie:FWsjNwgsnnoxwejtuv5hnilgcrzbzaeGnalorBnjecvbjxnzNnkwugarUazjkksotlIotditgf.JTkwUkqhzdybtygerrattksjzhnxsyeakwgesqiycgzhgovrkvkfaiozgszbtovrrrbtnzatvknxnotpfakltugrkhogggjbs.HnxktojcsjzegcdlwxxdgtFWsjNetaocsymhkmgfpuedrysrqkmhkdrdotwsgnqtvgelkntvguytne21fkqkgtarlrgcxlrafkcihnzrvsizxtutuvrkoerocdstmoltuvzuvarcbdaagizy.

Решение:
Для быстрого и качественного решения используем программу Cryptools:

 Вставляем наш текст в созданное окно:

Выбираем вкладку Analysis > Symmetric encryption (classic) > Ciphertext-only >  Vigenere

Далее программа находит длину ключа :
 
Жмём > Continue
Далее находит сам ключ:
 
Жмём> Decrypt , и получаем расшифрованный текст:

Расшифрованный текст:    SoutdernFederalUnivensityisamodernreoearchuniversitysithemphasisoninjovationsandentrapreneurship.Initoacademicactiviteesitcombinesstuzieswithfundamenpalandappliedsciance,aswellascutteng-edgetechnologeesandinnovativewpproaches.TheUnirersity’spositionenthenationalunirersityrankingspnovidespersuasivaevidencetoitsacdievements,apositeveimageandapasseonforexcellence:OFedUwasawardedtde5thplaceintheAnjualIndependentNwtionalUniversituRankings.SFedUeqqipsitsgraduatessithessentialskihlstogivethemacoipetitiveadvantacewhenitcomestogattingajob.TheknosledgeacquiredatOFedUenablesthempoboldlyfacethedamandsandchallencesofthe21stcenturuaswellastocontrebutetothedevelolmentofthelocalckmmunity.
Скачать программу можно здесь:
https://www.cryptool.org/en/ct1-downloads
Быстрая реализация на Python

Шифрование
m = "ATTACKATDAWN"  # исходное сообщение
k = "LEMON"  # ключ

k *= len(m) // len(k) + 1  # подгоняем ключ
c = ''.join([chr((ord(j) + ord(k[i])) % 26 + ord('A')) for i, j in enumerate(m)])  # шифруем
print(c) # LXFOPVEFRNHR

Расшифровывание
c = "LXFOPVEFRNHR"  # шифрованое сообщение
k = "LEMON"  # ключ

k *= len(c) // len(k) + 1  # подгоняем ключ
m = ''.join([chr((ord(j) - ord(k[i])) % 26 + ord('A')) for i, j in enumerate(c)])  # расшифровываем
print(m) # ATTACKATDAWN
```
