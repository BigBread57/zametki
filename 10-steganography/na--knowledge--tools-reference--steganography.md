# Steganography — tools reference

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / knowledge/tools-reference/steganography.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/knowledge/tools-reference/steganography.md)

---

# Steganography — tools reference

## Triage order by format

**Any file, first:**
```
file <f>; exiftool <f>; binwalk <f>       # metadata + appended-file check before assuming LSB
```

**Don't stop at a `Comment` field that looks like a red herring** ("nothing to
see here" and similar) — check `exiftool`'s own `Warning` field too. A
`Trailer data after ... IEND chunk` (PNG) or equivalent "data after end of
image" warning is `exiftool` directly telling you there's appended data past
the container's logical end — confirmed in practice on this corpus: the
Comment said "nothing to see here," the flag was sitting in the trailer the
Warning line pointed at. `xxd <f> | tail` or `strings` after the file's
expected end (after `IEND` for PNG) will show it directly.

**PNG:**
```
zsteg <f>                               # standard PNG/BMP LSB tool — try this first
zsteg -a <f>                              # exhaustive: all bit orders/channels, slower but thorough
pngcheck -v <f>                             # structural anomalies/extra chunks
.venv/bin/stegoveritas <f>                    # LSB, metadata, color-plane, appended-data checks in one pass — good second opinion
.venv/bin/python -c "
from PIL import Image
im = Image.open('<f>')
# manual LSB inspection: pull the low bit of each channel, look for a pattern
"
```

**JPEG:**
```
steghide extract -sf <f>            # try with no passphrase first, then common ones
steghide extract -sf <f> -p ""
stegseek <f> wordlists/Passwords/rockyou.txt    # cracks a steghide passphrase directly instead of guessing — try before hand-picking passphrases
```

**WAV / audio:**
```
.venv/bin/python -c "
import scipy.io.wavfile as wav, numpy as np
# spectrogram check — messages hidden in the frequency domain show up visually;
# plot with matplotlib and look for text/shapes
"
```

**Unknown / "just a file":**
```
binwalk -e <f>            # check for an appended archive/second file first — very common
strings -n 8 <f> | tail -50
```

## General discipline
Steganography challenges are almost always solvable by exhausting a short,
known list of techniques per format (above) rather than by cleverness — work
through the list for the actual format before assuming something exotic is
going on. `steghide`'s empty-passphrase attempt and `binwalk -e` catch a
surprising fraction of challenges by themselves.
