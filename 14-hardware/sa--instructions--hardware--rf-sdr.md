# RF / SDR — Capture, Demodulate, Replay

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/hardware/rf-sdr.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/hardware/rf-sdr.md)

---

---
name: rf-sdr
category: hardware
triggers: ["SDR", "RF", "433MHz", "315MHz", "keyfob", "garage door", "remote", "rtl_sdr", "hackrf", "gqrx", "GNU Radio", "URH", "rtl_433", "ASK", "OOK", "FSK", "IQ capture", "replay attack", ".cu8", ".wav signal"]
tools: ["rtl_sdr / rtl_433 / rtl-sdr", "HackRF (hackrf_transfer)", "gqrx", "GNU Radio Companion", "Universal Radio Hacker (URH)", "inspectrum", "SoapySDR"]
priority: medium
description: Capture, identify, demodulate, and replay ISM-band RF (433/315/868/915 MHz) — decode keyfobs/remotes and pull flags from signals.
---
# RF / SDR — Capture, Demodulate, Replay

## Trigger context
The challenge involves radio: a `.cu8`/`.cf32`/`.wav`/`.iq` capture file, a keyfob/garage/remote, or "listen on 433 MHz". Goal is to recover the transmitted data (a code, a flag, or to replay a command). Most CTF RF is simple ASK/OOK on the ISM bands.

## Objective
1. Find the carrier frequency and modulation (ASK/OOK vs FSK).
2. Demodulate to a bitstream / symbols.
3. Decode the protocol (or just recognize a known one via rtl_433).
4. Extract the flag from the payload, or build a replay to trigger the target.

## Methodology (decision tree)
1. IF given a capture file: identify its format/sample rate first, then jump to demodulation (URH / inspectrum). No radio needed.
2. IF live capture: SURVEY the band in gqrx to see where energy is (the fob transmits when pressed). Note center freq.
3. Try the KNOWN-PROTOCOL shortcut: run `rtl_433` — it auto-decodes hundreds of 433/315/868/915 devices. If it prints the data, you're done.
4. If unknown: CAPTURE IQ around the carrier, load into URH, identify modulation from the waveform, set symbol rate, and demodulate to bits.
5. INTERPRET bits: look for a preamble/sync, fixed vs rolling code, Manchester/PWM encoding. Convert to ASCII/hex -> flag.
6. If the task is to trigger something: REPLAY the recorded IQ (HackRF) or resynthesize from the decoded bits (URH generator / rpitx).

## Commands & payloads
```bash
# --- 0. Identify a given capture file ---
# .cu8 = 8-bit unsigned IQ (rtl_sdr default). .cf32/.complex = 32-bit float IQ.
# .wav can be audio (demodulated) or IQ. Sample rate is in the filename or brief.
file capture.iq ; ls -l capture.cu8

# --- 1. Known-protocol shortcut (do this first for 433/315/868/915) ---
rtl_433 -f 433.92M                       # live decode of known ISM devices
rtl_433 -A                               # analyzer mode: guesses modulation/timings of unknown
rtl_433 -r capture.cu8                   # decode a saved rtl_sdr file
rtl_433 -f 315M -s 1024k -F json         # 315 MHz (US fobs), JSON output

# --- 2. Survey the band (visual) ---
gqrx                                     # GUI: tune, watch waterfall, note the active freq
# or a quick power scan:
rtl_power -f 400M:450M:1k -g 40 -i 5 scan.csv   # find active channels in 400-450 MHz

# --- 3. Capture raw IQ ---
# RTL-SDR (receive only, cheap):
rtl_sdr -f 433.92M -s 2048000 -g 40 capture.cu8      # -s sample rate, -g gain
# HackRF (RX; can also TX for replay):
hackrf_transfer -r capture.cf32 -f 433920000 -s 8000000 -a 1 -l 32 -g 40
# Capture WHILE pressing the fob a few times so you get clean repeats.

# --- 4. Demodulate / analyze ---
# Universal Radio Hacker (best for CTF): interpret unknown ASK/OOK/FSK
urh                                      # GUI:
#  Interpretation tab: load capture -> it shows the demodulated square wave.
#  Set Modulation = ASK (OOK) or FSK; adjust "Samples/Symbol" until bits line up.
#  Analysis tab: it auto-groups messages; read the bitstream, toggle Manchester/
#  differential decoding, and export bits/hex.
# inspectrum (visual symbol measurement of an IQ file):
inspectrum capture.cu8                   # drag over a burst, read symbol rate/timing

# --- 5. Bits -> data ---
# In URH: right-click a column to mark preamble/sync/data; change display to ASCII/Hex.
# Manchester-encoded? URH decodes it; else de-Manchester by hand (01->1, 10->0).
# Search the decoded hex/ASCII for flag{...}.

# --- 6. Replay / transmit ---
# Straight IQ replay (fixed-code fobs) with HackRF:
hackrf_transfer -t capture.cf32 -f 433920000 -s 8000000 -a 1 -x 40
# Or regenerate from decoded bits in URH's Generator tab and transmit via HackRF.
# RTL-SDR cannot transmit; use HackRF, a 433 MHz TX module, or rpitx.

# --- Optional: GNU Radio Companion for custom flowgraphs (AM/FM/FSK demod chains) ---
gnuradio-companion                       # build: Source -> filter -> demod -> file/scope
```

## Branching logic
- `rtl_433` decodes it outright -> read the payload, extract flag; you're done, no manual demod.
- Waveform is on/off bursts (energy present/absent) -> ASK/OOK. Steady carrier that shifts -> FSK. Set URH modulation accordingly.
- Payload changes every press (rolling code, e.g. KeeLoq) -> replay of an old capture won't trigger; the challenge likely wants you to DECODE, not replay, or exploit a weak PRNG.
- Fixed code (same bits every press) -> replay works; capture once and retransmit.
- Signal is Manchester/PWM encoded -> toggle URH's decoding until bits are stable and periodic.
- Only a capture file, no hardware -> everything above works offline in URH/inspectrum/rtl_433 -r.

## Common pitfalls
- Sample rate mismatch on load: telling URH/rtl_433 the wrong rate garbles timing. Use the rate the file was captured at.
- Gain too high (clipping) or too low (buried in noise) -> unreadable. Tune `-g`; RTL-SDR sweet spot is often 30-45.
- Center-frequency DC spike sitting on the signal: offset-tune (capture 100-200 kHz off-center) so the signal isn't under the DC bin.
- Confusing samples-per-symbol with baud: in URH set samples/symbol so one bit = one plateau.
- Trying to TX with an RTL-SDR — it is receive-only.
- Legality: only transmit into the challenge's own hardware/attenuated setup; don't jam real bands.

## Proof / verification
- rtl_433/URH outputs a stable, repeating bitstream across multiple button presses (same payload = correct demod).
- Decoded ASCII/hex contains the flag, or the replay physically triggers the target (LED/relay/flag service).

## Cross-references
[hardware-triage](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--hardware-triage.md) [firmware-analysis](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--firmware-analysis.md) [side-channel-glitch](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--side-channel-glitch.md)

## Credible sources
- rtl_433 (merbanan): https://github.com/merbanan/rtl_433
- Universal Radio Hacker (URH): https://github.com/jopohl/urh
- RTL-SDR quick start & tutorials: https://www.rtl-sdr.com/rtl-sdr-quick-start-guide/
- HackRF docs (hackrf_transfer): https://hackrf.readthedocs.io/en/latest/
- Replay-attack tutorial (RTL-SDR + rpitx): https://www.rtl-sdr.com/tutorial-replay-attacks-with-an-rtl-sdr-raspberry-pi-and-rpitx/
