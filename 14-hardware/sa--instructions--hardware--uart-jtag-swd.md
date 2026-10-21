# UART / JTAG / SWD — Console and On-Chip Debug

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/hardware/uart-jtag-swd.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/hardware/uart-jtag-swd.md)

---

---
name: uart-jtag-swd
category: hardware
triggers: ["UART", "serial console", "TX RX", "USB-TTL", "screen", "picocom", "baud rate", "U-Boot prompt", "JTAG", "SWD", "OpenOCD", "JTAGulator", "halt CPU", "dump memory", "root shell"]
tools: ["USB-TTL adapter (CP2102/FT232/PL2303)", "screen / picocom / minicom", "logic analyzer (sigrok/Saleae)", "JTAGulator", "OpenOCD", "gdb / gdb-multiarch", "FT2232H / J-Link / ST-Link"]
priority: high
description: Get a serial console or on-chip debug — find UART/JTAG/SWD pins, drop to a root shell or U-Boot, halt the CPU and dump memory/flash.
---
# UART / JTAG / SWD — Console and On-Chip Debug

## Trigger context
A board (or [hardware-triage](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--hardware-triage.md) result) exposes a serial header or a debug port and you need interactive access: a shell, U-Boot, or a raw memory/flash dump. Two lanes: UART (easy, text console) and JTAG/SWD (deeper, halt-and-dump).

## Objective
- UART: identify TX/RX/GND, attach an adapter, read the boot log, get a shell (or interrupt U-Boot for a root boot).
- JTAG/SWD: identify the pinout, connect OpenOCD, halt the CPU, and dump internal flash/RAM (often where the flag or key lives on MCUs).

## Methodology (decision tree)
1. UART FIRST (lowest effort). Identify the 4 pins, connect, watch boot.
2. If a shell prompt appears with no/known password -> done, you have the target.
3. If it boots to a login you can't pass -> reboot and INTERRUPT U-Boot; set `init=/bin/sh` (or `single`) to get root without the password.
4. If no UART or UART is locked/authenticated -> go JTAG/SWD: identify pins (JTAGulator or known header), OpenOCD connect, `halt`, `dump_image` the flash/RAM.
5. On a Cortex-M with SWD, the whole internal flash is usually readable unless RDP/readout protection is set — if it is, pivot to [side-channel-glitch](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--side-channel-glitch.md) to glitch the readout-protection check.

## Commands & payloads
```bash
# ===================== UART =====================
# --- Identify pins (if unlabeled) ---
# GND: multimeter continuity to a known ground.
# VCC: steady 3.3V. TX: idles at VCC, dips during boot prints. RX: near VCC (pull-up), no dips.
# Or capture with a logic analyzer and decode (see hardware-triage.md).

# --- Auto-detect baud with a tool ---
pip install pyserial
python -m serial.tools.miniterm /dev/ttyUSB0 115200   # if garbage, try other bauds
# Baudrate brute helper:
baudrate.py -p /dev/ttyUSB0          # (devttys0 baudrate.py) sweeps common rates on live traffic

# --- Wiring: adapter GND->board GND, adapter RX->board TX, adapter TX->board RX.
#     Do NOT connect VCC. Match 3.3V logic.

# --- Attach a terminal ---
screen /dev/ttyUSB0 115200            # exit: Ctrl-a then k
picocom -b 115200 /dev/ttyUSB0        # exit: Ctrl-a Ctrl-x   (adds --imap lfcrlf if needed)
minicom -D /dev/ttyUSB0 -b 115200
# Common bauds: 115200, 57600, 38400, 19200, 9600. Readable text = right baud + right TX pin.

# --- Interrupt U-Boot for a root shell ---
# Reset the board with the terminal attached; SPAM a key (or Ctrl-C / specific key it names)
# during "Hit any key to stop autoboot".
=> printenv                            # dump env: look at bootargs, bootcmd, keys
=> setenv bootargs ${bootargs} init=/bin/sh    # boot straight to a root shell
=> boot
# Variants if init=/bin/sh is filtered: use `single`, or `init=/bin/bash`.
# Read raw memory/flash from U-Boot if no OS shell wanted:
=> md 0x80000000 40                    # memory display
=> sf probe && sf read 0x80000000 0 0x800000   # read 8MB SPI flash into RAM, then tftpput/dump
=> md.b 0x9f000000 0x100               # NOR flash mapped region

# ===================== JTAG / SWD =====================
# --- Identify pins with JTAGulator (unknown header) ---
# Wire every candidate pad to JTAGulator channels; connect to it at 115200:
screen /dev/ttyUSB0 115200
#  set target voltage (V), then:  "J" -> "I" runs JTAG IDCODE scan; "B" runs BYPASS scan
#  (finds TDI/TDO/TCK/TMS). "U" runs the UART scan (finds TX/RX + baud).
#  "S" scans for SWD (SWDIO/SWCLK) on newer firmware.
# Software-only alt (Arduino): JTAGenum brute-forces the same pins.

# --- OpenOCD connect (pick an interface + target cfg) ---
# SWD via ST-Link on an STM32:
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg
# JTAG via FT2232H (e.g. a generic FT2232 adapter) on a generic Cortex-M:
openocd -f interface/ftdi/ft2232h.cfg -c 'transport select jtag' -f target/... .cfg
# Generic Cortex-M SWD via CMSIS-DAP / J-Link similarly with interface/cmsis-dap.cfg / jlink.cfg
# Then in another shell talk to OpenOCD's telnet (port 4444):
telnet localhost 4444

# --- Halt and dump (in the OpenOCD telnet/console) ---
> reset halt                           # stop the CPU at reset
> halt
> flash banks                          # list flash banks/sizes
> dump_image firmware.bin 0x08000000 0x20000   # dump 128KB internal flash (STM32 base)
> mdw 0x20000000 16                     # peek RAM
> flash read_bank 0 dump.bin 0 0x40000  # alternative bank-relative read
# If readout protection blocks reads you'll get errors/zeros -> RDP set.

# --- GDB for scripted extraction ---
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg &   # gdb server on :3333
gdb-multiarch
(gdb) target extended-remote :3333
(gdb) monitor reset halt
(gdb) dump binary memory flash.bin 0x08000000 0x08020000
(gdb) x/64xw 0x20000000                 # examine RAM (search for flag/keys)
```

## Branching logic
- UART shell obtained -> loot live: `cat /etc/shadow`, `nvram show`, look for `flag`, then optionally dump flash from the running system (`dd if=/dev/mtdblock*`), pass images to [firmware-analysis](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--firmware-analysis.md).
- U-Boot reachable but env locked / `bootargs` protected -> try `sf read` to dump flash directly from U-Boot, or go JTAG/SWD.
- JTAG dump succeeds -> feed the `.bin` to [firmware-analysis](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--firmware-analysis.md) (it may be raw, not a filesystem — string/RE it).
- SWD present but RDP/readout protection enabled (reads return 0/error) -> [side-channel-glitch](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--side-channel-glitch.md) to glitch past the protection or the verify.
- No debug port at all, but an external SPI/I2C flash chip exists -> [spi-i2c-flash-dump](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--spi-i2c-flash-dump.md).

## Common pitfalls
- TX/RX not crossed: connect adapter RX to board TX (and vice versa). No output = swap them.
- Ground not common between adapter and board = garbage or nothing. Always tie GND.
- Wrong logic level (5V adapter on 3.3V board): corrupts data or damages the board — use a 3.3V/level-shifted adapter.
- Right baud but board TX idle-low convention (inverted UART, rare) -> looks like framing errors; try an inverting adapter.
- OpenOCD adapter speed too high on JTAG -> unstable IDCODE. Lower with `adapter speed 100` (kHz).
- Assuming JTAG dump is a filesystem: MCU flash is usually a bare firmware image; use RE tooling, not `unsquashfs`.

## Proof / verification
- UART: readable boot log and a shell prompt that runs commands.
- U-Boot root: `id` shows uid=0 / you can read `/etc/shadow`.
- JTAG/SWD: `dump_image`/`dump binary memory` produces a non-zero, non-constant file whose `strings` shows real code/data.

## Cross-references
[hardware-triage](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--hardware-triage.md) [firmware-analysis](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--firmware-analysis.md) [spi-i2c-flash-dump](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--spi-i2c-flash-dump.md) [side-channel-glitch](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--side-channel-glitch.md)

## Credible sources
- OpenOCD User's Guide (flash & general commands): https://openocd.org/doc/html/Flash-Commands.html
- JTAGulator (Joe Grand): https://www.grandideastudio.com/jtagulator/ and https://github.com/grandideastudio/jtagulator
- UART-to-root walkthrough: https://exfil.co/2019/02/14/uart-to-root-the-harder-way/
- U-Boot bootargs / init=/bin/sh recovery: https://help.endian.com/hc/en-us/articles/218146638
