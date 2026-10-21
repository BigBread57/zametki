# SPI / I2C Flash Dump — Reading the Chip Directly

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/hardware/spi-i2c-flash-dump.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/hardware/spi-i2c-flash-dump.md)

---

---
name: spi-i2c-flash-dump
category: hardware
triggers: ["SPI flash", "25Q64", "W25Q128", "SOIC-8", "SOIC clip", "24C02", "AT24", "I2C EEPROM", "flashrom", "CH341A", "Bus Pirate", "FT2232", "chip-off", "dump flash", "read eeprom"]
tools: ["flashrom", "CH341A programmer", "Bus Pirate", "FT2232H (ftdi_spi)", "SOIC-8 test clip (Pomona 5250)", "Raspberry Pi (spidev)", "i2c-tools (i2cdetect/i2cdump)", "sigrok (SPI/I2C decode)", "binwalk"]
priority: high
description: Identify and dump SPI NOR / I2C EEPROM flash chips (in-circuit or chip-off) with flashrom, then carve the firmware.
---
# SPI / I2C Flash Dump — Reading the Chip Directly

## Trigger context
[hardware-triage](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--hardware-triage.md) found an external memory chip — an SOIC-8 SPI NOR (25-series: `W25Q`, `MX25L`, `EN25`, `GD25`) or a small I2C EEPROM (24-series: `24C02..24C512`, `AT24`, `24LC`). Reading it directly often yields the whole firmware/flag with no console needed. This is frequently the FASTEST path.

## Objective
Produce a complete, correct binary dump of the flash, verify it, then extract the flag/filesystem. Prefer in-circuit (clip) first; fall back to chip-off (desolder) if the board interferes.

## Methodology (decision tree)
1. IDENTIFY the chip and its protocol from the top marking + pin 1 dot.
   - `25xx` (8 pins) = SPI NOR -> flashrom lane.
   - `24xx` (8 pins) = I2C EEPROM -> i2c lane (flashrom does not do I2C EEPROMs; use i2c-tools or a Bus Pirate).
2. Note the DENSITY from the marking (e.g. `W25Q64` = 64 Mbit = 8 MB) so you know the expected dump size.
3. Choose ACCESS: in-circuit with an SOIC-8 clip (try first), or chip-off (desolder to a socket/adapter) if the host SoC holds the bus or power fights the clip.
4. WIRE the programmer (CH341A / Bus Pirate / FT2232). Power the chip at its rated voltage (3.3V; some are 1.8V — check the datasheet, a 1.8V chip on 3.3V is destroyed).
5. DUMP twice, compare — a good read is reproducible. Then verify size == density.
6. CARVE with binwalk -> [firmware-analysis](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--firmware-analysis.md).

## Commands & payloads
```bash
# ================= SPI NOR (25-series) with flashrom =================
# --- Detect the chip (auto JEDEC ID) ---
flashrom -p ch341a_spi                       # CH341A USB programmer
flashrom -p buspirate_spi:dev=/dev/ttyUSB0   # Bus Pirate
flashrom -p ft2232_spi:type=232H,port=A      # FT2232H (e.g. an FTDI mini-module)
flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=8000   # Raspberry Pi SPI header
# If it prints a chip name/size, detection worked.

# --- Read (dump). If the chip is DETECTED but ambiguous, force -c ---
flashrom -p ch341a_spi -r dump1.bin
flashrom -p ch341a_spi -r dump2.bin
flashrom -p ch341a_spi -c "W25Q64.V" -r dump.bin      # force chip model if needed
sha256sum dump1.bin dump2.bin                # MUST match -> reliable read
ls -l dump.bin                               # size should equal density (8MB for W25Q64)

# --- CH341A note: fixed 2MHz SPI, 3.3V. Many CH341A boards ship at 5V VCC on the
#     clip — MOD or verify it's 3.3V or you kill 3.3V-only chips. It powers the chip;
#     with the SoC also on the bus, in-circuit reads can be noisy -> chip-off.

# --- Bus Pirate manual SPI sniff/read alternative (if flashrom lacks the chip):
#     enter SPI mode, send 0x03 <24-bit addr> then clock out bytes.

# ================= I2C EEPROM (24-series) =================
# --- Raspberry Pi / Linux i2c-tools ---
i2cdetect -y 1                               # find the address (24C02 typ. 0x50-0x57)
i2cdump -y 1 0x50                            # dump a small (<=256B) EEPROM to screen
# Full read of larger 24xx (16-bit addressing) — loop reads with i2ctransfer:
for a in $(seq 0 255); do i2cget -y 1 0x50 $a b; done   # crude 256-byte read
# Better for >256B: i2ctransfer for 2-byte addressing, or a Bus Pirate/eeprog:
eeprog -f -x -r 0:0x800 /dev/i2c-1 0x50 > eeprom.bin     # read 2KB from 24C16
# --- Bus Pirate I2C: sniff a live bus (macro) to catch reads without desoldering.

# ================= Sniff instead of read (in-circuit, non-invasive) =================
# Capture the SPI/I2C bus with a logic analyzer at boot; decode with sigrok:
sigrok-cli -d fx2lafw -c samplerate=24m --samples 24M \
  -C D0,D1,D2,D3 -o spi.sr                    # CLK,MOSI,MISO,CS
sigrok-cli -i spi.sr -P spi:clk=D0:mosi=D1:miso=D2:cs=D3 -A spi   # decode transactions
# I2C decode: -P i2c:scl=D0:sda=D1 -A i2c   (recover the address + bytes)

# ================= Carve the dump =================
binwalk -Me dump.bin                          # -> _dump.bin.extracted/  then firmware-analysis.md
strings -n 8 dump.bin | grep -iE 'flag\{|password|BEGIN .*PRIVATE'
```

## Branching logic
- flashrom prints "No EEPROM/flash device found" -> bad clip contact, wrong pinout (rotate the clip), chip held by SoC (hold SoC in reset or go chip-off), or wrong voltage.
- flashrom says "Multiple flash chip definitions match" -> pick with `-c "<exact model>"`.
- In-circuit read is inconsistent between dumps -> the SoC is contending on the bus. Tie the SoC reset low, or desolder the chip (chip-off) onto a SOIC-8 adapter.
- Chip is 1.8V (e.g. some MX25U / `25U` series) -> use a 1.8V-capable programmer or adapter; never 3.3V.
- Dump obtained -> feed to [firmware-analysis](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--firmware-analysis.md). If it's a bare MCU image not a filesystem, treat as raw firmware / RE.
- Only an I2C EEPROM present (tiny) -> it usually holds config/serial/secret, not a filesystem; string it directly.

## Common pitfalls
- Pin 1 orientation: the clip's red wire / the chip's dot must align. Reversed = no ID or garbage.
- CH341A 5V trap: stock boards often present 5V to the chip; 3.3V-only NOR can be damaged. Verify with a multimeter first.
- Forgetting to hold the host SoC in reset for in-circuit reads — the SoC drives CS/CLK and corrupts the dump.
- Reading only once: always dump twice and hash-compare; clip contact is flaky.
- Size mismatch (dump smaller than density) means truncated read/wrong chip def — fix `-c` or contact before analyzing.
- Trying to use flashrom for a 24-series I2C EEPROM — flashrom is SPI/parallel/LPC/FWH, not I2C EEPROM; use i2c-tools/eeprog.

## Proof / verification
- Two dumps hash-identical AND size == expected density = trustworthy image.
- `binwalk` shows real structures (uImage/SquashFS) or `strings` shows coherent config/flag.

## Cross-references
[hardware-triage](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--hardware-triage.md) [firmware-analysis](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--firmware-analysis.md) [uart-jtag-swd](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--uart-jtag-swd.md)

## Credible sources
- flashrom CH341A programmer docs: https://flashrom.org/supported_hw/supported_prog/ch341ab.html
- flashrom CLI manpage (programmers/`-r`/`-c`): https://flashrom.org/classic_cli_manpage.html
- Bus Pirate + flashrom SPI: http://dangerousprototypes.com/docs/SPI
- sigrok SPI/I2C protocol decoders: https://sigrok.org/wiki/Protocol_decoders
