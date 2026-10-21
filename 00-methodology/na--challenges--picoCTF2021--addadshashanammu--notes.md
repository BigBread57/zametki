# Notes — Tab, Tab, Attack

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / challenges/picoCTF2021/addadshashanammu/notes.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/challenges/picoCTF2021/addadshashanammu/notes.md)

---

# Notes — Tab, Tab, Attack

## 2026-08-06

- Ran `tools/triage_file.sh` on `Addadshashanammu.zip` (5166 bytes,
  sha256 `095df26cb59e322171b99742066676edbc3ee6ae9ae096be203b4dbf8b7ca2f6`).
- `binwalk` showed the zip contains 7 nested directories, each named after
  a long fake-Akkadian-sounding word, matching the challenge's "long
  directory structures" flavor text — the whole point being tab-completion
  through `Addadshashanammu/Almurbalarammi/Ashalmimilkala/
  Assurnabitashpi/Maelkashishi/Onnissiralis/Ularradallaku/` rather than
  typing it by hand.
- Deepest directory contains two files: `fang-of-haynekhtnamet.c` (source,
  96 bytes) and `fang-of-haynekhtnamet` (compiled ELF, 16712 bytes).
- **Flag found directly in `strings` output during triage**, no further
  steps needed — `fang-of-haynekhtnamet.c`'s single `printf` call has the
  flag in plaintext.
- Extracted (`unzip`) and read the source directly to confirm — verbatim
  match, not guessed:
  ```c
  #include <stdio.h>
  int main(){
  printf("*ZAP!* picoCTF{l3v3l_up!_t4k3_4_r35t!_fc588427}\n");
  }
  ```
- Did not execute the compiled binary — unnecessary once the source
  confirmed the flag, and per project policy on not running unknown
  binaries without need.

## Flag

`picoCTF{l3v3l_up!_t4k3_4_r35t!_fc588427}` — verbatim substring of the
extracted `.c` source, matches the `picoCTF{...}` format.


---

## Зависимости и ссылки вне библиотеки

Следующие файлы или ресурсы упомянуты в исходнике, но не входят в эту библиотеку документов. Их наличие и работоспособность здесь не проверялись.

- `challenges/Addadshashanammu.zip`
- `challenges/picoCTF2021/addadshashanammu/evidence/Addadshashanammu.zip`
- `tools/triage_file.sh`
