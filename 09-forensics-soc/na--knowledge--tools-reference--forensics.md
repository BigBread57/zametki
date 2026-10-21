# Digital forensics — tools reference

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / knowledge/tools-reference/forensics.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/knowledge/tools-reference/forensics.md)

---

# Digital forensics — tools reference

## Metadata first, always
```
exiftool <file>
```
Cheapest possible step, run it before anything else on any provided file —
GPS coordinates, author names, software versions, and even flags show up here
directly often enough that it's worth doing unconditionally.

## File carving
```
binwalk <file>              # identify embedded/appended file signatures
binwalk -e <file>            # extract them
foremost -i <file> -o out/     # broader carving by file-type signature, good for disk images
scalpel -c /etc/scalpel/scalpel.conf -o out/ <file>   # needs a config enabling the types you want first
```
Order: `binwalk` (fast signature scan) → if it finds something, `-e` extract →
if it finds nothing but you suspect carving is needed anyway (disk image, raw
dump), `foremost`/`scalpel` as a broader sweep.

## Memory forensics
```
.venv/bin/vol -f <image> windows.info        # identify the OS/profile first
.venv/bin/vol -f <image> windows.pslist       # then pick plugins based on what you're looking for
.venv/bin/vol -f <image> windows.cmdline
.venv/bin/vol -f <image> linux.bash           # bash history, for Linux images
```
Always run `windows.info`/`linux.banners` first — volatility3 needs to identify
the profile before most other plugins are useful, and it tells you which plugin
family (windows.* vs linux.*) applies.

## Network captures
```
tshark -r <cap>.pcap                                    # quick overview
tshark -r <cap>.pcap -Y "http.request"                     # filter to a protocol/condition
tshark -r <cap>.pcap --export-objects http,out/               # pull transferred files out directly
tshark -r <cap>.pcap -Y "tcp.stream eq 3" -T fields -e data     # follow one stream's raw bytes
```
`--export-objects` is the fastest path to "there's a file hidden in this pcap" —
try it before manually following streams.

## Disk / filesystem images
```
file <image>              # confirm it's actually a filesystem/disk image and what kind
mount -o loop,ro <image> /mnt/x   # if it's a mountable filesystem (needs root)
testdisk <image>                    # partition recovery
```
