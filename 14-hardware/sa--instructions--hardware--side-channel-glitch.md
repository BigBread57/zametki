# Side-Channel & Fault Injection — Power Analysis and Glitching

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/hardware/side-channel-glitch.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/hardware/side-channel-glitch.md)

---

---
name: side-channel-glitch
category: hardware
triggers: ["side channel", "power analysis", "DPA", "CPA", "SPA", "fault injection", "glitch", "voltage glitch", "clock glitch", "ChipWhisperer", "recover AES key", "bypass PIN", "secure boot bypass", "timing attack", "power trace"]
tools: ["ChipWhisperer (Husky/Lite/Nano/Pro)", "chipwhisperer Python API", "NewAE tooling", "oscilloscope", "crowbar MOSFET glitcher", "numpy", "Jupyter"]
priority: medium
description: Recover keys via power analysis (SPA/DPA/CPA) and bypass checks via voltage/clock fault injection with ChipWhisperer.
---
# Side-Channel & Fault Injection — Power Analysis and Glitching

## Trigger context
The challenge hints at physical leakage or fault injection: "recover the AES key from power traces", a provided trace set (`.npy`/`traces`), a PIN/password check to bypass, or a secure-boot/readout-protection gate you couldn't beat via [uart-jtag-swd](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--uart-jtag-swd.md). Two lanes: PASSIVE (power analysis to extract a key) and ACTIVE (glitch to skip a check).

## Objective
- Power analysis: correlate power consumption with a key-dependent intermediate (AES S-box output) to recover the key byte-by-byte (CPA), or read a value directly off the trace (SPA).
- Glitching: find timing + strength parameters that fault the target exactly when it evaluates a check, so a wrong PIN/signature is accepted or a memory-readout lock is skipped.

## Methodology (decision tree)
1. CLASSIFY: given traces to analyze offline -> CPA/DPA lane. Physical target to fault -> glitch lane.
2. POWER ANALYSIS (CPA on AES):
   a. Get N traces each with known plaintext + the sampled power during the first round.
   b. For each key-byte guess (0-255), model the leakage = Hamming weight of `Sbox(plaintext ^ keyguess)`.
   c. Correlate the model vector against every time sample across all traces; the guess with the highest |correlation| peak is the correct key byte.
   d. Repeat for all 16 bytes -> full key. ~50 clean traces often suffice for software AES.
3. SPA shortcut: if the operation branches visibly (e.g. an RSA square-vs-multiply, or a byte-by-byte password memcmp that returns early), read the secret straight off one/few traces.
4. GLITCHING (fault a check):
   a. Locate the target instant (trigger on I/O edge, or scan an offset window around the check).
   b. Sweep glitch PARAMETERS: offset (when), width/repeat (how long), and for voltage, the crowbar timing. Grid-search.
   c. Classify each attempt: NORMAL / CRASH (reset) / SUCCESS (check bypassed). The exploitable window is the narrow band between mostly-crash and mostly-normal.
   d. Automate the sweep, log outcomes, converge on the success cluster.

## Commands & payloads
```python
# ============ CPA on AES (offline, numpy — works on any trace set) ============
import numpy as np
traces = np.load('traces.npy')        # shape (N, samples)
pt     = np.load('plain.npy')         # shape (N, 16)  known plaintexts
sbox   = np.array([...], dtype=np.uint8)   # standard AES S-box (256 entries)
HW = np.array([bin(x).count('1') for x in range(256)])   # Hamming-weight table

key = []
for b in range(16):                   # recover each key byte independently
    best_guess, best_corr = 0, 0.0
    for g in range(256):
        model = HW[sbox[pt[:, b] ^ g]]           # predicted leakage per trace
        # Pearson correlation of model vs each time sample:
        m = model - model.mean()
        num = (m[:, None] * (traces - traces.mean(0))).sum(0)
        den = np.sqrt((m**2).sum() * ((traces - traces.mean(0))**2).sum(0))
        corr = np.abs(num / den)
        peak = corr.max()
        if peak > best_corr:
            best_corr, best_guess = peak, g
    key.append(best_guess)
print("recovered key:", bytes(key).hex())

# ============ ChipWhisperer capture + CPA (real hardware) ============
import chipwhisperer as cw
scope = cw.scope()                    # auto-detects CW-Lite/Husky/Nano
target = cw.target(scope, cw.targets.SimpleSerial)
scope.default_setup()
scope.adc.samples = 5000
project = cw.create_project("aes_cpa", overwrite=True)
ktp = cw.ktp.Basic()
for i in range(100):
    key_, text = ktp.next()
    trace = cw.capture_trace(scope, target, text, key_)   # sends PT, grabs power
    project.traces.append(trace)
# Run the built-in CPA analyzer:
import chipwhisperer.analyzer as cwa
attack = cwa.cpa(project, cwa.leakage_models.sbox_output)
results = attack.run()
print(results.find_maximums())        # best key-byte guesses + correlation

# ============ Clock / voltage glitch parameter sweep (ChipWhisperer) ============
scope.glitch.clk_src   = "clkgen"
scope.glitch.output    = "clock_xor"     # clock glitching (use "glitch_out" crowbar for VCC)
scope.glitch.trigger_src = "ext_single"  # fire relative to the target's trigger
gc = cw.GlitchController(groups=["success","reset","normal"],
                         parameters=["width","offset","ext_offset"])
gc.set_range("width",  -48, 48)
gc.set_range("offset", -48, 48)
gc.set_range("ext_offset", 0, 500)       # cycles after trigger = WHEN the check runs
for glitch_setting in gc.rand(num_steps=2000):
    scope.glitch.width  = glitch_setting[0]
    scope.glitch.offset = glitch_setting[1]
    scope.glitch.ext_offset = glitch_setting[2]
    scope.arm()
    target.simpleserial_write('p', bytes(16))   # trigger the guarded operation
    scope.capture()
    resp = target.simpleserial_read('r', 16, timeout=50)
    if target_reset_detected(): gc.add("reset")
    elif check_bypassed(resp):  gc.add("success"); print("GLITCH:", glitch_setting)
    else:                       gc.add("normal")
# Plot the success cluster (offset vs ext_offset) to home in on parameters.
```

## Branching logic
- Given traces only -> pure numpy/ChipWhisperer-Analyzer CPA; no hardware. If CPA fails, check alignment (traces must be time-aligned; resync on a feature) or try a Hamming-DISTANCE model / last-round attack (hardware AES).
- Recovered key byte correlations low/ambiguous -> capture/collect more traces, improve trigger alignment, or attack a different intermediate (S-box output vs MixColumns).
- Glitch target: no `success` ever, only `normal`/`crash` -> widen/shift `ext_offset` (you're missing the check instant); adjust glitch strength (width) between the crash edge and the normal edge.
- Secure boot / RDP readout-lock is the gate -> glitch the moment it verifies/branches, then immediately dump via [uart-jtag-swd](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--uart-jtag-swd.md) before it re-locks.
- PIN check with early-exit timing -> a pure TIMING attack (measure response time per digit) may recover it with no glitching at all.

## Common pitfalls
- Unaligned traces destroy CPA correlation — align on a sharp common feature (trigger, a distinct spike) before correlating.
- Wrong leakage model: software AES leaks Hamming WEIGHT of S-box output; hardware AES often leaks Hamming DISTANCE of register updates (attack the last round).
- Too few / too noisy traces -> false key peaks. Average repeated measurements; increase N.
- Glitch too strong = permanent crash/brick-reset; too weak = no effect. The exploitable setting is a narrow band — search finely near the crash boundary.
- Missing the trigger instant: without a good trigger you must sweep `ext_offset` across the whole window — slow but necessary.
- Not resetting/power-cycling the target between glitch attempts -> stale state poisons results.

## Proof / verification
- CPA: recovered key decrypts a known ciphertext to its plaintext, or matches the flag/expected key. A correct byte shows one guess with a clearly dominant correlation peak.
- Glitch: the guarded action succeeds reproducibly at the found parameters (wrong PIN accepted, memory now readable, boot continues) — reproduce it several times to confirm it's not a fluke.

## Cross-references
[uart-jtag-swd](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--uart-jtag-swd.md) [firmware-analysis](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--firmware-analysis.md) [hardware-triage](https://raw.githubusercontent.com/BigBread57/zametki/main/14-hardware/sa--instructions--hardware--hardware-triage.md)

## Credible sources
- ChipWhisperer documentation: https://chipwhisperer.readthedocs.io/
- ChipWhisperer tutorials (SCA101 CPA on AES, glitching labs): https://github.com/newaetech/chipwhisperer-jupyter
- NewAE ChipWhisperer project: https://github.com/newaetech/chipwhisperer
- CPA/DPA background (Riscure/Ledger Donjon writeups): https://github.com/Ledger-Donjon/lascar
