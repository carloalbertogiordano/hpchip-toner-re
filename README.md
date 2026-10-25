# HP Toner Chip — Hardware Reverse Engineering

Hardware security research on the memory chip ("E-label", HP's internal term)
embedded in an **HP 135A / 135X** (W1350A / W1350X) laser toner cartridge, for
the **HP LaserJet MFP M234** family (M209/M234/M236). The goal was to understand
how the chip is built, how it stores data, and whether it communicates in a way
that could be read or reset — with the practical secondary goal of resetting the
toner-level/usage counter on the **original** chip.

Work done on a cartridge I own, as personal hardware-security research (right-to-
repair context). **No printer was available** during the work, which shapes the
conclusions below.

---

## TL;DR

A **rigorous negative result.** Using only a multimeter, an Arduino UNO R4 WiFi,
and a breadboard, I characterized the chip electrically and ruled out — *with my
own positive/negative controls* — every category of communication physically
possible on a 2-contact, antenna-less device: digital voltage-level protocols
(1-Wire-style), current modulation on the power line, RFID/NFC, and a purely
passive element. The original chip is a **genuine active IC** (ESD-diode network
confirmed) that stayed silent to every stimulus tried. The most useful outcome
was **methodological**: three separate "promising" signals turned out to be
artifacts — including one that was 50 Hz mains hum — and were caught by
controls rather than mistaken for discoveries.

## Hardware under test

| | |
|---|---|
| Printer (family) | HP LaserJet MFP M234dw / M234dwe / M234sdne / M234sdw (M209/M234/M236) |
| Cartridge | HP 135A / 135X (W1350A / W1350X) |
| Original chip | Chip-on-board (COB), black epoxy glob-top, **2 exposed pads**, embedded in the cartridge body; marking abraded/illegible |
| Clone chip | Compatible aftermarket chip, **4 pads** (2 large + 2 small), likely **Apex Microelectronics** (Zhuhai); non-functional on the printer even before testing |
| Instrumentation | Arduino UNO R4 WiFi, multimeter (diode/Ω/V), CH341A, limited resistor set (470 Ω, combined in series/parallel) |

## Method & findings

### 1. Physical identification
Pad count confirmed (2 original / 4 clone). The original is a COB glob-top
bonded into the structural plastic — **decapsulation was ruled out** (requires
lab failure-analysis tooling, too destructive for this setup). Chip markings on
both were unreadable and matched nothing in public databases.

### 2. Pinout mapping (multimeter, diode mode)
Systematic pair-wise probing showed the original conducts bidirectionally with
similar values (~1076–1280 mV) — not a clean single-diode signature, consistent
with an **internal ESD-protection network** (diodes to internal VDD and GND).
Conclusion: a real IC, not a passive resistive/fuse element. On the clone, a
12-reading map produced consistent rail/signal hypotheses (one pad behaving as
GND/VSS, one as VDD-like, two as pure signal pins).

### 3. Active electrical tests (Arduino: OneWire + raw probe)
Two sketches: a standard **OneWire** ROM-search scan, and a **raw probe** that
bypasses the library, sends reset pulses of varying length (60–1000 µs) and
samples the line at ~1 µs resolution to catch any activity independent of
standard 1-Wire timing. All driven through an external pull-up at **3.3 V**
(never direct 5 V) with a series protection resistor.

- **Original chip:** no OneWire response, no raw-probe activity; electrically
  *clean* — the line rises normally under pull-up, no anomaly, simply no
  response to the protocol/timing attempted.
- **Clone chip:** no response either, but with an **electrical anomaly** — the
  line never exceeds ~0.6–0.7 V regardless of pull-up strength (down to ~117 Ω
  equivalent, ~28 mA), polarity, or small-pad state. The near-constant voltage
  vs. varying current is the signature of a **diode in conduction**, pointing to
  a damaged protection diode rather than a protocol mystery (consistent with the
  clone already not working on the printer before testing).

### 4. ICSP attempt on the clone
Initial hypothesis: the two small pads = PIC ICSP (PGC/PGD), target PIC12F683.
Checked against Microchip's documentation: **Low-Voltage Programming entry for
the 12F683 needs MCLR pulled low plus VDD/VSS/PGC/PGD — five connections, not
four.** With only 4 pads, standard PIC ICSP doesn't fit directly; untested
4-wire alternatives remain (I²C debug, SWD if it's an ARM Cortex-M0, UART
bootloader).

### 5. Extended timing & current-modulation hypothesis — with controls
- **Slow-wake** (10–300 ms soak before reset, 20 ms sample window): no activity
  on the original at any soak time.
- **Current modulation** (sense resistor + differential ADC read on A0/A1 to
  estimate instantaneous current): an initial ~3.6 mA "signal" **did not
  reproduce** under a dummy-load control and a repeat with the chip reconnected
  (~0.02–0.05 mA, indistinguishable from a passive load). No real current
  modulation.

### 6. Three false positives (the methodological core)
Three separate results initially *looked* like real chip activity and were each
disproven by controls:
1. **150–212 transitions/cycle** after a wiring fix → not reproducible on a
   fresh wiring layout (unstable breadboard contact).
2. The **~3.6 mA current range** above → gone under dummy-load and repeat.
3. A **structured "burst" pattern** that appeared only on cold reconnect, even
   reproduced across two physical connections → reappeared **with the chip
   fully disconnected** (floating wire only). Inter-burst gaps summed to
   ~19.7–19.9 ms (**~50 Hz**, Italian/European mains) and ~100 Hz harmonics:
   it was **mains hum** picked up by a high-impedance floating line, not the
   chip. (The apparent polarity dependence was explained too: the internal
   diode anchors the resting level nearer or farther from the logic threshold,
   so identical ambient noise crosses the threshold often in one orientation,
   rarely in the other.)

## What the public record says
- **No public technical documentation** for the HP 135A/135X chip (pinout,
  protocol, voltage) in electronics forums, GitHub, HP patents, or the clone
  maker's site. HP's internal name for these is **"E-label."**
- A related service manual (M402/M403) shows a **3.3 V** dominant logic rail
  (no +5 V for logic), reinforcing the 3.3 V assumption, and states toner level
  is sensed *optically* — **empirically contradicted on the M234**, where the
  reported level followed the chip, not an independent optical measure.
- **HP Dynamic Security** is a cryptographic challenge-response between printer
  and cartridge chip; the secret is in the chip's internal flash and is not
  extractable by a simple memory dump by design.
- A known system-side vector (not chip-side) is **downgrading the printer
  firmware** below the Dynamic Security enforcement — relevant to the practical
  counter-reset goal, untested here.

## Conclusion
With only 2 contacts and no antenna, the chip must communicate via a
voltage-level protocol — the alternatives were excluded with controls. Its
silence doesn't prove it can't; it shows that a stimulus generated with passive
resistors and software-timed Arduino pulses doesn't reproduce precisely enough
(slew rate, tight timing, low-impedance drive) what a real printer controller
generates. Closing the gap needs either a **logic-analyzer capture of real
printer↔chip traffic**, or **analysis of the official printer firmware** (where
the code that generates this signal is in the clear) — the planned next step.

## Methodological takeaway
Three "structured" or "consistent" results were not evidence of signal. Each was
settled by: repeatability over time without touching the circuit; a negative
control (same test, component removed or replaced by a known load); and, for
periodic patterns, checking the frequency against known ambient noise (mains).
That discipline is the point of this project as much as any single measurement.

## Status / next steps
- Logic-analyzer capture of a real communication cycle (needs a compatible
  printer, even briefly).
- I²C scan on the clone's small pads (alternative to PIC ICSP).
- Obtain a second, known-good clone as a clean electrical reference.
- Evaluate the printer-firmware-downgrade path for the practical counter reset.

## Ethics & scope
Performed on a cartridge I own, for research and right-to-repair purposes. No
cryptographic secret was extracted and none is published. HP Dynamic Security is
the subject of ongoing right-to-repair debate (FTC, EU).

## License

See [`LICENSE`](LICENSE).
