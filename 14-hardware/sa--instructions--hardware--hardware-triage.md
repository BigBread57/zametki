# Hardware Triage — Reading a Board and Choosing a Path

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/hardware/hardware-triage.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/hardware/hardware-triage.md)

---

---
name: hardware-triage
category: hardware
triggers: ["PCB photo", "board photo", "chip markings", "debug header", "test pads", "unknown board", "identify interface", "hardware challenge", "which pins", "logic analyzer", "multimeter"]
tools: ["eyes/vision (read the photo)", "search engines + datasheet sites", "multimeter", "sigrok-cli / PulseView", "Saleae Logic 2", "JTAGulator", "flashrom"]
priority: high
description: First-look triage of a PCB photo or physical board — ID chips, find debug/interface pins, and route to the correct sub-playbook.
---
# Hardware Triage — Reading a Board and Choosing a Path

## Trigger context
A challenge gives a PCB photo, a physical board, or a description mentioning chip markings, headers, test pads, or "which pins". This file is the ENTRY POINT: it does not solve the challenge, it classifies the board and dispatches to a sub-playbook. Always run triage before touching wires.

## Objective
Produce, from a photo or a board, a structured inventory:
1. What is the main processor / SoC (arch: ARM, MIPS, RISC-V, Xtensa)?
2. Where does firmware live (external SPI/I2C flash, eMMC, internal)?
3. What debug/comms interfaces are exposed (UART, JTAG, SWD, SPI, I2C)?
4. Which unpopulated headers / test pads reach those interfaces?
Then hand off to the right playbook with concrete pin candidates.

## Methodology (decision tree)
1. INVENTORY CHIPS. Zoom the photo. Read every top-line marking on every IC. For each, note package (SOIC-8, TQFP, BGA, QFN) and rough size.
   - Largest chip with many pins = main SoC/MCU. Search the marking for a datasheet.
   - SOIC-8 / SOP-8 near the SoC with a marking like `25Q64`, `W25Q128`, `MX25L`, `EN25` = SPI NOR flash -> [spi-i2c-flash-dump](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--spi-i2c-flash-dump.md).
   - SOIC-8 marked `24C02/04/08/16`, `AT24`, `24LC` = I2C EEPROM (small, config/secrets) -> [spi-i2c-flash-dump](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--spi-i2c-flash-dump.md).
   - `BGA` marked `KLM`, `THGBM`, `EMMC`, or `H26M` = eMMC (needs chip-off or ISP).
2. IDENTIFY ARCH from the SoC datasheet (drives emulation/toolchain later): MIPS/ARM = Linux router-class -> [firmware-analysis](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--firmware-analysis.md); Cortex-M / ESP = MCU, likely on-chip flash -> [uart-jtag-swd](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--uart-jtag-swd.md) + [side-channel-glitch](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--side-channel-glitch.md).
3. FIND DEBUG HEADERS / PADS. Look for:
   - A row of 3-6 unpopulated 0.1" holes near the SoC = classic UART (often labeled `J1`, `TP`, `CON`, or silkscreen `RX/TX/GND`). -> [uart-jtag-swd](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--uart-jtag-swd.md)
   - A 2x5 (10-pin) or 2x7 (14-pin) or 2x10 (20-pin) header = JTAG. A 2x5 tiny 1.27mm = ARM Cortex SWD/JTAG. -> [uart-jtag-swd](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--uart-jtag-swd.md)
   - 4 pads in a cluster of 2x2 near a Cortex-M = SWD (SWDIO, SWCLK, GND, VCC/RST).
4. IF NO LABELS: you must probe. Go to the pin-identification procedure below, then dispatch.
5. RECORD everything (chip list, candidate pin functions, arch) before wiring anything.

## Commands & payloads
```bash
# --- Datasheet / marking lookup (do this per chip marking) ---
# Search: "<exact top marking>" datasheet   e.g. "W25Q128JV datasheet"
# Useful sources: datasheetspdf.com, alldatasheet.com, the vendor site.
# Flash chips: consult the flashchips list / SPI JEDEC ID once dumped.

# --- Continuity / voltage with a multimeter (power OFF for continuity) ---
# 1. Continuity (beep) mode: find GND — any pad that beeps to a known ground
#    (USB shield, electrolytic cap negative, large copper pour).
# 2. DC volts, board POWERED: VCC pads read a steady 3.3V (or 1.8V).
#    UART TX idles HIGH at VCC and dips as it prints boot text; RX ~ VCC via pull-up.

# --- Logic analyzer capture with sigrok (cheap FX2 clone or real HW) ---
sigrok-cli --scan                              # list detected devices
sigrok-cli -d fx2lafw --samples 8M -c samplerate=4m \
  -C D0,D1,D2,D3 -o cap.sr                      # capture 4 unknown pads at boot
# Open cap.sr in PulseView; add the UART decoder, set baud (see below), read text.
# For an unknown baud: capture the TX line, measure the NARROWEST pulse width (s);
# baud ~= 1 / narrowest_pulse. Common: 115200, 57600, 38400, 9600.

# PulseView (GUI): Decoders -> UART (data=chosen channel). Or from CLI:
sigrok-cli -i cap.sr -P uart:rx=D0:baudrate=115200 -A uart

# --- Saleae Logic 2 alternative: capture, add Async Serial analyzer, auto-baud. ---

# --- If pins are totally unknown and header looks like JTAG/UART: JTAGulator ---
# Wire all candidate pads to JTAGulator channels, then run its UART and JTAG scans
# (see uart-jtag-swd.md).
```

## Branching logic
- SPI/I2C flash chip present + goal is "extract firmware/flag" -> [spi-i2c-flash-dump](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--spi-i2c-flash-dump.md) (fastest: dump the chip directly, no console needed).
- UART header found + running Linux SoC -> [uart-jtag-swd](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--uart-jtag-swd.md) to get a shell, then [firmware-analysis](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--firmware-analysis.md) on whatever you pull.
- You already have a `.bin` / firmware image (no physical board) -> skip straight to [firmware-analysis](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--firmware-analysis.md).
- Cortex-M / secure-boot / "bypass the PIN check" framing -> [uart-jtag-swd](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--uart-jtag-swd.md) (SWD dump) and [side-channel-glitch](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--side-channel-glitch.md) (glitch the check).
- RF remote / keyfob / 433MHz / antenna on board -> [rf-sdr](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--rf-sdr.md).

## Common pitfalls
- Do NOT inject VCC into a TX/RX line — identify VCC/GND first and never drive them. Common the GND of your adapter to the board GND.
- 5V USB-TTL on a 3.3V (or 1.8V) board can fry it. Match logic levels; use a level shifter or a 3.3V adapter.
- The largest chip is not always the flash; RAM (DDR) is also big — RAM markings look like `NT5`, `K4B`, `MT41`, `H5TQ`. Don't waste time reading DDR.
- A "UART" header may have TX/RX swapped on silkscreen. If you see no output, swap TX<->RX.
- Test pads under a sticker or shield: remove/peek before concluding there is no debug port.

## Proof / verification
- Chip ID confirmed when the datasheet package + pin count matches the photo.
- UART confirmed when PulseView/Saleae decodes readable ASCII boot log (e.g. `U-Boot`, `Linux version`).
- Interface routed correctly when the sub-playbook's first step succeeds (e.g. flashrom detects the SPI chip).

## Cross-references
[uart-jtag-swd](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--uart-jtag-swd.md) [spi-i2c-flash-dump](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--spi-i2c-flash-dump.md) [firmware-analysis](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--firmware-analysis.md) [side-channel-glitch](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--side-channel-glitch.md) [rf-sdr](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--rf-sdr.md)

## Credible sources
- sigrok/PulseView supported hardware & decoders: https://sigrok.org/wiki/Supported_hardware
- JTAGulator (Joe Grand) — pin ID overview: https://www.praetorian.com/blog/jtagulator-vs-jtagenum-tools-for-identifying-jtag-pins-in-iot-devices/
- OWASP Firmware Security Testing Methodology (FSTM): https://github.com/scriptingxss/owasp-fstm
- flashrom supported chips/programmers: https://flashrom.org/supported_hw/index.html
