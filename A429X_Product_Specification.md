# A429X — Secure, Self-Diagnosing ARINC 429 Controller IP

## Product Specification

| Field | Value |
|---|---|
| Document ID | A429X-PS-001 |
| Revision | 0.4 (Draft for review, aligned with v1.0 RTL) |
| Date | 2026-10-05 |
| Status | Specification of the v1.0 RTL (implemented and verified in simulation; ZCU106 on-board self-test 32/32 PASS) |
| Target devices | Vendor-neutral RTL. Reference platform is the AMD Zynq UltraScale+ MPSoC on the ZCU106 (XCZU7EV) |
| Classification | Company Confidential — product definition |

### Revision history

| Rev | Date | Description |
|---|---|---|
| 0.1 | 2026-10-05 | First full draft covering the core, SLP, LHM, register map, verification and the ZCU106 plan |
| 0.2 | 2026-10-05 | Aligned with the v1.0 RTL: 16-byte MAC header, sync body format, TIME units, CMAC bound, FREEZE semantics, MAD-based fingerprint scale, register-map details, implementation status (§20) |
| 0.3 | 2026-10-05 | Security hardening and traceability review. Changes: <br>• Zeroization scope and glitch filter [KEY-005] <br>• Fail-secure sealing [SLP-034] <br>• Secure-bus default [KEY-003] <br>• Seal check order and recovery [SLP-041] <br>• Exact verdict layout [SLP-043] <br>• T_SEAL_MAX and K_ALARM defaults <br>• Error-injection set [TX-015] <br>• RX_RATE_MEAS unit and refresh-scan clock floor <br>• KEYMAP/CRYPTO_UTIL addresses, TX_CNT_BLOCKED, SEAL_BLOCKED and NONSEC_KEY bits <br>• Measured CMAC time (390 clk), area and Fmax (§13.2) <br>• Updated §20 |
| 0.4 | 2026-10-05 | Verification closure: seven new benches, defects fixed, ambiguities resolved. Changes: <br>• Rx: GLITCH_MIN default scales with clock [RX-001], FRAME_ERR counts events [RX-003], auto-rate on the 4-bit mean with defined windows [RX-005], latency budget notes [RX-008], bus-load window boundary [RX-010], META bit 15 MBX_VALID <br>• Time: PPS latency compensated, TIME_PPS_VAL registers [TIME-002] <br>• SLP: FV exhaustion event timing and stopped-group behaviour [SLP-010], persistence and leap wording [SLP-011], sync and epoch timing [SLP-020, SLP-031], KID sampling [KEY-007], TIME sample point <br>• LHM/FP: RATE_PPM sign and source [LHM-011], WARN/ALARM and score rules [LHM-030/031], feature vector and measured detection [FP-001], baseline restore [FP-002], D_ema [FP-003], ZT-04 at +300 ppm <br>• Tx: TX-004 edge rule, scheduler tick range and tick 0 [TX-011], FREEZE staging [TX-012/013], counters cleared by any write <br>• Outputs registered and reset-gated with the clock stopped [CLK-002]; LOOP_INT is off-bus [SAF-004] <br>• Constant multiplies as shift-add: −20 DSP (§13.2) <br>• ZCU106 on-board self-test 32/32 PASS (§20) <br>• §20 results updated |

---

## Table of contents

1. Introduction
2. Problem statement and value proposition
3. Product overview
4. ARINC 429 baseline core (compliant functions)
5. Sealed Label Protocol (SLP): in-band authentication that legacy receivers ignore
6. Line Health Monitor (LHM) and Physical Fingerprint IDS
7. Timebase and time distribution
8. External interfaces and parameters
9. Register map
10. Interrupts and event log
11. Clocking, reset and CDC
12. Safety, reliability and certification support
13. Performance and resource targets
14. Software deliverables
15. Verification plan
16. ZCU106 evaluation platform and lab test plan
17. Product deliverables and licensing
18. Roadmap
19. Risks, open issues and legal notes
20. v1.0 RTL implementation status

Appendix A: SLP worked example and test-vector format.
Appendix B: SLP profile descriptor.
Appendix C: Glossary.

---

## 1. Introduction

### 1.1 Purpose

This document specifies **A429X**, a soft IP core for FPGAs and ASICs. A429X implements the ARINC 429 digital information transfer system in full and adds two capabilities that the ARINC 429 standard does not provide:

1. **SLP (Sealed Label Protocol).** It adds cryptographic authentication, replay protection and strong integrity to an existing ARINC 429 bus. It needs no new wiring. Legacy receivers on the same bus keep working unchanged.
2. **LHM (Line Health Monitor) with a Physical Fingerprint IDS.** It measures the electrical and timing health of each link continuously, so wiring and driver degradation is predicted before data errors start. It also detects rogue or substituted transmitters from their physical-layer signature.

This document serves as:
- the product definition for sales and marketing,
- the top-level Hardware Requirements source for RTL development (each "shall" has a traceable ID, in support of DO-254 / ED-80),
- the basis for the verification plan and the ZCU106 reference design.

### 1.2 Scope

In scope: the digital controller (RTL), its register interface, its integration interfaces, the reference drivers, the verification environment and the evaluation platform.

Out of scope: the analog ARINC 429 line driver and line receiver. These are external devices. The IP specifies the logic-level interface to them and gives recommendations for the evaluation card. System-level key management (key generation, distribution and crypto-officer roles) is also out of scope. The IP provides the hooks that system key management uses.

### 1.3 Reference documents

| Ref | Document |
|---|---|
| [R1] | ARINC Specification 429 Part 1: Functional Description, Electrical Interface, Label Assignments and Word Formats (latest supplement) |
| [R2] | ARINC Specification 429 Part 2: Discrete Word Data Standards |
| [R3] | ARINC Specification 429 Part 3: File Data Transfer Techniques |
| [R4] | RTCA DO-254 / EUROCAE ED-80: Design Assurance Guidance for Airborne Electronic Hardware; AMC 20-152A / AC 20-152A |
| [R5] | RTCA DO-326A / ED-202A (Airworthiness Security Process), DO-356A / ED-203A (Security Methods) |
| [R6] | FIPS 197 (AES); NIST SP 800-38B (CMAC); IETF RFC 4493 (AES-CMAC test vectors) |
| [R7] | NIST SP 800-108r1 (KDF); NIST SP 800-38F (Key Wrap); NIST SP 800-232 (Ascon) |
| [R8] | AMBA AXI and ACE Protocol Specification (AXI4, AXI4-Lite); AMBA AXI4-Stream |
| [R9] | AMD UG1244: ZCU106 Board User Guide |
| [R10] | AUTOSAR Specification of Secure Onboard Communication (SecOC). Informative: SLP uses its truncated-MAC and truncated-freshness design pattern |

> ARINC specifications are copyrighted by SAE ITC. This document restates electrical and timing values only where the IP needs them. Customers and developers must hold their own licensed copies of [R1]–[R3]. Where the two disagree, [R1] is normative.

### 1.4 Conventions

- **shall** = mandatory requirement (traceable, verified). **should** = recommendation. **may** = option.
- Requirement IDs have the form `[AREA-NNN]`, for example `[TX-012]`.
- **ARINC bit numbering:** ARINC bit *k* (1…32) maps to register bit *k−1*. ARINC bit 1 is transmitted first.
- **Label bit order:** in ARINC 429 the label (bits 1–8) is sent MSB first and all other fields are sent LSB first. With `GCTRL.LABEL_NAT = 1` (the default), software reads and writes the label as its natural octal value in register bits [7:0], and the core reverses the bits on the wire. With `LABEL_NAT = 0`, register bits [7:0] hold the label in wire order.
- Octal labels are written `0oNNN`.
- `clk` = core clock (`aclk`). Times given in "clk" units scale with `G_CLK_HZ`.

---

## 2. Problem statement and value proposition

### 2.1 What ARINC 429 does not solve

ARINC 429 is still the most widely installed avionics data bus. It is in almost every commercial transport aircraft, many business jets, helicopters and military transports, and in retrofit programs. It was designed in the late 1970s, and it has these gaps:

| # | Gap in ARINC 429 | Consequence today |
|---|---|---|
| G1 | **No source authentication.** Any device that can drive the pair is trusted. | DO-326A/ED-202A security assessments now have to treat physical access threats, maintenance laptops and compromised LRUs. There is no in-protocol countermeasure. |
| G2 | **No replay protection or sequence numbers.** | Recorded traffic can be replayed. Lost or injected words cannot be detected. |
| G3 | **Weak integrity.** A single odd-parity bit detects only odd numbers of bit errors per word. | Double-bit errors pass silently. There is no detection across a sequence of words. |
| G4 | **No data age or time context.** | Each consumer implements freshness monitoring in software, differently on every program. |
| G5 | **No physical-layer diagnostics.** A link is either "working" or "failed". | Chafing, corrosion, connector fretting and driver aging stay invisible until parity errors or loss of data appear. This drives intermittent faults and "No Fault Found" (NFF) LRU removals, one of the largest avionics maintenance cost drivers. |
| G6 | **No rogue-transmitter detection.** Simplex buses assume a single transmitter, but nothing enforces it. | A tap, a substituted LRU or a back-fed device can inject words during inter-word gaps without being detected. |

The industry's usual answer is to move to ARINC 664 (AFDX) or other networks. On in-service fleets this means rewiring and recertifying the aircraft, which is usually uneconomical.

### 2.2 The A429X answer

**A429X makes an existing ARINC 429 wire authenticated, tamper-evident and self-diagnosing. Only the LRUs that need the protection are changed. Wiring, connectors and legacy receivers stay as they are.**

| Gap | A429X feature | How it works |
|---|---|---|
| G1, G2, G3 | **SLP**: Sealed Label Protocol (§5) | The transmitter periodically sends a short "seal" of standard ARINC 429 words on a system-assigned spare label. The seal carries a truncated AES-CMAC over the protected words and a monotonically increasing freshness value. Legacy receivers ignore the unknown label. A429X receivers verify, quarantine and report. |
| G4 | Hardware timestamps, mailbox data age and a per-label refresh monitor (§4.6, §7). SLP sync seals can optionally carry authenticated time (§5.8). | Every received word carries a 64-bit timestamp. Every label has min and max refresh limits that are checked in hardware. |
| G5 | **LHM** (§6) | Each word is measured for bit-rate offset, pulse widths, jitter, slew, amplitude margin and noise, at three front-end tiers ranging from standard comparators to an ADC. Statistics, trends and a health score come out of the hardware. |
| G6 | **Physical Fingerprint IDS** (§6.6) and the **Tx Readback Monitor** (§6.7) | The receiver learns the transmitter's electrical "signature". The clock-offset signature works even with standard comparators. Words whose signature does not match are flagged one at a time. A transmitter that sees activity on its line during its own gaps raises an alarm. |

### 2.3 Design principles

- **[GEN-001] Backward compatibility.** Every word that A429X puts on the wire shall be a standard-conformant ARINC 429 word: 32 bits, correct parity, legal timing, gaps of at least 4 bit times. A429X shall need no change to any legacy receiver beyond the system ICD reserving the seal label(s).
- **[GEN-002] Opt-in granularity.** SLP and LHM shall be selectable per channel, and SLP per (label, SDI). A program can start in observation mode (monitor-only) and move to enforcement later.
- **[GEN-003] Determinism.** All A429X timing shall be bounded and documented: latency, crypto scheduling and seal insertion. There shall be no unbounded queues in the data path.
- **[GEN-004] Certifiability.** The RTL shall follow DO-254 DAL A design practice (§12). Security functions shall be separable, so that the baseline core can be certified on its own.
- **[GEN-005] Vendor neutrality.** The RTL shall be synthesizable on AMD, Microchip (rad-tolerant/flash), Intel/Altera and Lattice/Efinix devices without vendor primitives. Memories shall be inferred. Optional vendor wrappers may add ECC BRAM.

---

## 3. Product overview

### 3.1 Feature summary

**Baseline core (ARINC 429 Part 1 compliant):**
- 1–16 transmit and 1–16 receive channels, each set independently to high speed (100 kbps), low speed (12.0–14.5 kbps) or a custom rate.
- A fractional NCO bit-rate generator: ≤ 0.01 % rate error at any core clock from 20 to 250 MHz.
- Odd parity generation and checking. Even and transparent parity for test.
- Tx FIFO, a 256-entry periodic **hardware scheduler**, and timed transmit at an absolute time.
- Rx label/SDI filter (1024 entries), a FIFO with 64-bit timestamps and metadata, per-label **mailboxes**, and a per-label **refresh-rate monitor** (stale and too-fast detection).
- Full error detection: parity, word length, gap, bit rate, glitches, invalid line state.
- Rx automatic rate detection. Bus-load measurement.
- **Error injection** on Tx for test-equipment use: parity, word length, gap, rate skew and glitches.
- A Bulk Transfer Assist for implementing ARINC 429 Part 3 file transfer.
- AXI4-Lite control. Optional AXI4-Stream data paths for DMA.

**SLP option (security):**
- AES-128-CMAC (mandatory algorithm). Ascon (NIST SP 800-232) is a build-time option.
- Up to 4 seal groups per Tx channel. Selective protection per (label, SDI).
- Truncated tags of 21, 42 or 63 bits. A 48-bit freshness value with 13-bit truncation on the wire.
- Receiver delivery modes: Monitor, Early-Release and Enforce (verified-only).
- Sync seals for resynchronization, with optional authenticated time.
- A key store (16 × 128-bit keys) holding write-only keys, with a KDF, zeroization and power-up known-answer self-test.

**LHM option (diagnostics and intrusion detection):**
- Front-end tiers FE0 (standard comparator), FE1 (dual-threshold margin comparators) and FE2 (ADC samples).
- 16 metrics with min, max, mean and variance; counters; a trend ring buffer; and a health score with WARN and ALARM thresholds.
- Physical Fingerprint IDS that learns a baseline and flags anomalies per word and per window.
- Tx Readback Monitor: wrap-around compare, foreign-activity detection and driver health.

### 3.2 License tiers (commercial packaging)

| Tier | Contents | Typical customer |
|---|---|---|
| **A429X-Core** | Baseline ARINC 429 core, drivers and DO-254 kit for the core | LRU vendors, test equipment |
| **A429X-Secure** | Core + SLP + key store | Programs with DO-326A findings, defense, retrofit |
| **A429X-Sense** | Core + LHM + Fingerprint IDS + Readback | MRO, airlines (predictive maintenance), test equipment |
| **A429X-Complete** | All features | New LRUs, security-focused retrofits |
| **A429X-Lab** (add-on) | Virtual Line Emulator, Attacker Emulator, analyzer firmware | Test-equipment vendors, internal validation |

### 3.3 Top-level block diagram

```
                     +--------------------------------------------------------------------------------+
 s_axil (AXI4-Lite)<>| Register File & Address Decode | Interrupt Ctrl & Event Log | Timebase (64-bit) |<- pps_in / ext time
 m_axis_rx         <-| AXIS Rx Merge (opt)            | Key Store + Zeroize        | Crypto Engine     |<- zeroize_n
 s_axis_tx         ->| AXIS Tx Demux (opt)            |   (16 x 128b, write-only)  | AES-CMAC (shared) |
 irq               <-|                                                                                |
                     |  TX CHANNEL [n]                                                                |
                     |   Tx FIFO ----+                                                                |
                     |   Scheduler --+--> Arbiter --> SLP Sealer --> Error Inject --> RZ Encoder/NCO --+--> tx_hi/tx_lo/tx_slope/tx_en
                     |   Bulk Assist-+     ^  (seal > sched > fifo)      ^                             |
                     |                     |                             | (crypto req)                 |
                     |   Readback Monitor + LHM-RB <-----------------------------------------------------+--- rb_hi/rb_lo (+margin)
                     |                                                                                |
                     |  RX CHANNEL [n]                                                                |
 rx_hi/rx_lo     --->|   Front-End Adapter --> Glitch Filter --> Bit Decoder --> Word Checker -->      |
 rx_hi_m/rx_lo_m --->|   (FE0/FE1/FE2)   |                        (auto-rate)    (parity/len/gap)    |
 s_axis_adc[n]   --->|                   v                                              |              |
                     |            LHM Metrics --> Stats/Trend --> Fingerprint IDS       v              |
                     |                                                     Label Table Lookup         |
                     |                                                     SLP Verifier + Quarantine  |
                     |                                                     Mailbox + Refresh Monitor  |
                     |                                                     Rx FIFO / AXIS             |
                     +--------------------------------------------------------------------------------+
```

---

## 4. ARINC 429 baseline core

### 4.1 Word format (informative summary of [R1])

| ARINC bits | Field | Notes |
|---|---|---|
| 1–8 | Label | Octal identifier, transmitted MSB first |
| 9–10 | SDI | Source/Destination Identifier (or data, depending on the label) |
| 11–29 | Data | BNR / BCD / discrete / maintenance. Bit 29 is the sign for BNR |
| 30–31 | SSM | Sign/Status Matrix |
| 32 | P | Odd parity over bits 1–31 |

### 4.2 Electrical and timing reference (informative; normative source is [R1])

These values constrain the external PHY and the configurable timing windows of the IP.

| Parameter | High speed | Low speed |
|---|---|---|
| Bit rate | 100 kbps ± 1 % | 12.0–14.5 kbps (one rate per bus) |
| Bit time | 10 µs ± 2.5 % | 1/R ± 2.5 % |
| High (pulse) time | 5 µs ± 5 % | (1/R)/2 ± 5 % |
| Rise / fall (10–90 %) | 1.5 ± 0.5 µs | 10 ± 5 µs |
| Inter-word gap | ≥ 4 bit times | ≥ 4 bit times |
| Max word rate (4-bit gap) | 2,777 words/s (360 µs/word) | ≈ 347 words/s at 12.5 kbps |
| Tx differential output HI / NULL / LO | +10 ± 1 V / 0 ± 0.5 V / −10 ± 1 V | same |
| Tx output impedance | 75 ± 5 Ω, balanced | same |
| Rx decision | HI > +6.5 V, LO < −6.5 V, NULL within ±2.5 V | same |
| Receivers per bus | ≤ 20 | ≤ 20 |

### 4.3 Transmitter requirements

- **[TX-001]** Each Tx channel shall encode words as bipolar return-to-zero on two logic outputs, `tx_hi` (drive HI) and `tx_lo` (drive LO). Both deasserted means NULL.
- **[TX-002]** The core shall never assert `tx_hi` and `tx_lo` at the same time, including during reset, reconfiguration and SEU events. A hardware interlock that is independent of the encoder FSM shall enforce this.
- **[TX-003]** The bit rate shall come from a 32-bit phase-accumulator NCO producing half-bit ticks: `INC = round(2^32 · 2 · f_bit / f_clk)`. Presets: HS = 100 kbps, LS = 12.5 kbps. Custom rates shall be programmable through `TX_NCO_INC`.
- **[TX-004]** Average bit-rate error shall be ≤ 0.01 % for 20 MHz ≤ f_clk ≤ 250 MHz. Each edge shall lie within 1 clk of the NCO's ideal edge time (a whole number of half bits after the previous edge), so edge placement jitter is ≤ 1 clk. The residual INC rounding error is a rate error (within the 0.01 %), not jitter.
- **[TX-005]** The inter-word gap shall be programmable from 4 to 255 bit times (`TX_CTRL.GAP`). Values below 4 shall be accepted only when `TX_CTRL.TEST_MODE = 1`.
- **[TX-006]** Parity modes: ODD (default; bit 32 generated), EVEN (test) and TRANSPARENT (bit 32 taken from software).
- **[TX-007]** A `tx_slope` output shall select the line-driver slew (0 = HS, 1 = LS). It follows `TX_CTRL.RATE` unless overridden.
- **[TX-008]** A `tx_en` output shall enable the external driver. It shall be deasserted while the channel is disabled or in reset, and the line shall then be held at NULL.
- **[TX-009]** Tx FIFO: depth `G_TX_FIFO_DEPTH` (16–1024). Status flags EMPTY, THRESH and FULL. A write while FULL shall be dropped, counted, and raise `TX_OVF`.
- **[TX-010]** Timed transmit: when `TX_FIFO_CTL.TIMED = 1`, the next pushed word shall carry a 64-bit release time. The FIFO head shall wait until `TIME ≥ release`. Release accuracy shall be ≤ 1 bit time plus any word already in progress.
- **[TX-011]** Hardware scheduler: up to `G_SCHED_ENTRIES` (0/64/256) entries. Each entry holds `{DATA[31:0], EN, PERIOD[11:0], OFFSET[11:0]}`, with period and offset in `SCHED_TICK` units (100 µs to 10 ms recommended; any value 1–65535 µs is accepted and 0 reads as 1; default 1 ms). An entry fires when `(tick − OFFSET) mod PERIOD == 0`, where tick 0 is the first tick after the scheduler is enabled or `SCHED_CTRL.RESYNC`.
- **[TX-012]** Software shall be able to update scheduler `DATA` with a single 32-bit write. The scheduler shall never send a torn word. While `SCHED_CTRL.FREEZE = 1`, fired entries are queued but not transmitted (one queued instance per entry; further fires count as `SCHED_LATE`). The arbiter stages one word ahead, so at most one word already staged when FREEZE is set is still transmitted, with the value it held when staged. DATA is read when a word is staged, so every word started after FREEZE is cleared carries the new value; a fire batch interrupted by FREEZE completes after the clear with new values. Software sets FREEZE, updates a multi-label set, and clears FREEZE, so no word of the set goes out mid-update.
- **[TX-013]** Arbitration priority shall be SLP seal words > scheduled words > FIFO words. `TX_CTRL.PRIO_FIFO` reverses the order of the last two. A word in progress is never pre-empted. Because of the one-word staging, a ready higher-priority word can wait for the word in progress plus one already-staged word.
- **[TX-014]** If a scheduled entry cannot be sent within one PERIOD of becoming ready, the core shall set `SCHED_LATE` and count it. Bus overload must be visible.
- **[TX-015]** Error injection (only with `TEST_MODE = 1`, per word through `TX_FIFO_CTL.INJECT` code and 7-bit parameter; ignored otherwise):
  - 1: wrong parity
  - 2: 31-bit word
  - 3: 33-bit word
  - 4: short gap of 1–3 bits
  - 5: bit-rate skew of parameter/512, signed, from −12.5 % to +12.3 %
  - 6: a glitch of 8, 16, 32 or 64 clk (parameter [6:5]) in the NULL half of bit parameter [4:0]
  - 7: HI/LO swap on bit parameter [4:0]
- **[TX-016]** Counters: words sent, seal words sent, FIFO overflows, scheduler late events. All 32-bit, saturating, cleared by any write (the value written is ignored).

### 4.4 Receiver requirements

- **[RX-001]** Inputs `rx_hi` and `rx_lo` (FE0) shall pass through 2-FF synchronizers and then a glitch filter. A level shall be accepted only after it has been stable for `RX_CTRL.GLITCH_MIN` × 8 clk. The reset value of GLITCH_MIN scales with `G_CLK_HZ` to ≈ 500 ns (6 at 100 MHz, 1 at 20 MHz), which suits HS; for LS, software should program ≈ 2 µs (25 at 100 MHz).
- **[RX-002]** Bit decode: a qualified HI pulse decodes as 1 and a qualified LO pulse as 0. The receiver needs no clock recovery because RZ is self-clocking.
- **[RX-003]** Word framing: a NULL interval longer than `GAP_DET` (default 1.75 bit times, range 1.25–3.5) shall end the word. A word with fewer or more than 32 bits shall be flagged `FRAME_ERR`. With `WAIT_GAP = 0` the word is delivered at the end of bit 32. A 33rd bit then raises the `FRAME_ERR` interrupt and counter, but it cannot mark the word already delivered. With `WAIT_GAP = 1` the word itself is flagged ([RX-008]). The `FRAME_ERR` counter and interrupt count framing events, not bits: one per word, so a merged 64-bit burst counts once.
- **[RX-004]** Rate checking: pulse width and bit period shall be checked against windows set by `RX_CTRL.TOL` (1–15 %, default 5 %). A violation shall flag `RATE_ERR` without discarding the word unless filtering asks for that.
- **[RX-005]** Auto-rate (`RATE = AUTO`): the receiver shall average the bit periods of the first 4 bits after each gap (fewer if the word is shorter) and classify the bus as HS when the mean period is 0.875–1.14 × P_HS (≈ 87.7–114.3 kbps), LS when it is 0.8125–1.094 × P_LS (≈ 11.4–15.4 kbps, covering 12.0–14.5 kbps), or otherwise UNKNOWN. `RX_RATE_MEAS` shall report the last measured bit period in clk. A change shall raise `RATE_CHANGE`.
- **[RX-006]** Parity: bit 32 shall be checked (odd) when `PARITY_CHK = 1`. Failures flag `PARITY_ERR`.
- **[RX-007]** `rx_hi` and `rx_lo` both asserted for longer than GLITCH_MIN shall flag `INVALID_STATE` (receiver or line fault) and count it.
- **[RX-008]** A word shall be available to the label lookup ≤ 2 µs after the trailing edge of the bit-32 pulse, plus the fingerprint verdict time when the LHM is enabled: ≤ 3 µs per pending word in the shared LHM engine, worst case NUM_RX × 3 µs at 100 MHz (verdict timeout 655 µs, after which the word is delivered without a verdict). If `RX_CTRL.WAIT_GAP = 1`, the core waits for gap confirmation first (detects 33-bit words; adds GAP_DET latency). The 2 µs includes the glitch filter at its reset value. The end-of-learn fingerprint baseline computation runs in idle engine time and yields to pending words, so it does not extend these bounds.
- **[RX-009]** Timestamp: each word shall get a 64-bit timebase value taken at the leading edge of bit 1. The fixed synchronizer and filter delay shall be compensated, giving ±2 clk accuracy relative to the filtered input.
- **[RX-010]** Bus load: `RX_LOAD` shall report the percentage of time the bus carries word activity (bits plus mandatory 4-bit gap) over a programmable window (10 ms–10 s). Resolution is 0.1 %. A word counts toward the window in which its last bit completes, so a window can be off by one word (up to 3.6 % for a 10 ms HS window).

### 4.5 Label filter and routing

- **[RX-020]** Each Rx channel shall hold a label table of 1024 × 32-bit entries, indexed by `{SDI[1:0], LABEL_NAT[7:0]}`. When `RX_CTRL.SDI_FILT = 0`, entries for SDI = 0 apply to all SDI values.
- **[RX-021]** Entry format:

| Bits | Field | Meaning |
|---|---|---|
| 0 | ACCEPT | Store word in Rx FIFO / AXIS |
| 1 | MBX | Update mailbox |
| 2 | IRQ | Raise `LABEL_MATCH` on reception |
| 3 | PROT | Word is in an SLP Protected Label Set |
| 5:4 | GRP | SLP seal group (0–3) when PROT = 1 |
| 6 | STREAM | Route to AXIS instead of the register FIFO |
| 7 | RSVD | 0 |
| 19:8 | MAX_INT | Max refresh interval in `RATE_TICK` units (0 = no stale check) |
| 31:20 | MIN_INT | Min refresh interval (0 = no too-fast check) |

- **[RX-022]** With `RX_CTRL.FILTER_EN = 0`, all words shall be accepted to the FIFO. SLP classification still uses the table.
- **[RX-023]** Words with errors shall be stored only when `RX_CTRL.STORE_ERR = 1`. They shall never update a mailbox.

### 4.6 Mailboxes and refresh-rate monitor

- **[RX-030]** Mailbox (if `G_MAILBOX ≠ 0`): one 16-byte entry per (label, SDI): `{DATA, META, TS_LO, TS_HI}`. The latest valid word overwrites the entry. `G_MAILBOX = 1` gives 256 entries (SDI ignored) and `= 2` gives 1024 entries.
- **[RX-031]** Mailbox reads shall be coherent. Reading `DATA` latches the `META` and `TS` shadow registers for that entry.
- **[RX-032]** Refresh monitor: a background scanner shall visit every entry with `MAX_INT ≠ 0` at least once per 100 µs. It shall set `META.STALE` and raise `STALE` if `now − TS > MAX_INT`. The `MAX_INT` used is the value captured with the entry's last update, so a label-table change takes effect at the next reception. A word arriving earlier than `MIN_INT` after the previous one shall set `META.FAST` and raise `TOO_FAST` (a babbling-transmitter indicator). The scan takes 4 clk per mailbox entry. The 100 µs bound therefore holds for f_clk ≥ 41 MHz with 1024 entries, or ≥ 10.3 MHz with 256.
- **[RX-033]** In SLP ENFORCE mode, mailboxes for protected labels shall be updated only by verified words (§5.10).

### 4.7 Rx FIFO entry

- **[RX-040]** Rx FIFO depth `G_RX_FIFO_DEPTH` (16–4096). Each entry is 128 bits: `DATA[31:0]`, `META[31:0]`, `TS[63:0]`. Reading `RX_FIFO_DATA` pops the entry and latches `META` and `TS` into shadow registers. When full, new words shall be dropped and counted, and the next stored entry shall carry `META.OVF_BEFORE = 1`.
- **[RX-041]** META format (also used for mailboxes and AXIS):

| Bits | Field |
|---|---|
| 0 | PARITY_ERR |
| 1 | FRAME_ERR |
| 2 | GAP_ERR |
| 3 | RATE_ERR |
| 6:4 | AUTH: 0 NONE, 1 PENDING, 2 PASS, 3 FAIL_TAG, 4 FAIL_COUNT, 5 FAIL_REPLAY, 6 FAIL_TIMEOUT, 7 UNSYNCED |
| 7 | FP_ANOMALY (fingerprint mismatch on this word) |
| 8 | LHM_WARN (margin miss or slew violation on this word) |
| 9 | SEAL_WORD (entry is an SLP seal word; only with `STORE_SEAL = 1`) |
| 10 | OVF_BEFORE |
| 11 | VERDICT (entry is an SLP verdict record, §5.10) |
| 12 | STALE (mailbox only) |
| 13 | FAST (mailbox only) |
| 14 | RSVD |
| 15 | MBX_VALID (mailbox reads only: the entry has been written since reset or flush) |
| 31:16 | SEQ: per-channel receive sequence number (wraps) |

### 4.8 Bulk Transfer Assist (for ARINC 429 Part 3)

- **[BT-001]** Tx: a block of up to 4096 words in host memory (AXIS/DMA) or in the FIFO can be sent back-to-back at the minimum gap. A completion interrupt follows.
- **[BT-002]** Rx: a capture window shall store every word whose label falls inside a programmable label range into a dedicated stream, with the count and a timeout.
- **[BT-003]** The Part 3 protocol logic (Williamsburg/BOP handshakes, file framing) is a software library delivered with the driver. The hardware only guarantees ordering and gap control. *A full hardware Part 3 engine is a roadmap item (§18).*

---

## 5. Sealed Label Protocol (SLP)

### 5.1 Concept

SLP turns a sequence of ordinary ARINC 429 words into an **authenticated epoch**. The transmitter collects the words it sends on labels in a *Protected Label Set* (PLS). After *N* protected words, or after a time limit, it sends a **seal**: a short burst of standard ARINC 429 words on a dedicated **seal label** `L_S`. The seal contains:
- a header with the key ID, the epoch word count and the low 13 bits of a 48-bit **freshness value** (FV), and
- 1–3 **tag words** holding a truncated AES-CMAC over the header context and every protected word in the epoch, in order.

```
time ─────────────────────────────────────────────────────────────────────────────►
 │ W 0o203 │ W 0o206 │ W 0o270 │ W 0o210 │ ... │ W 0o203 ║ SHW │ TW1 │ TW2 ║ W 0o203 │ ...
     P         P         U         P               P      ╚═ seal on L_S ═╝     P
 P = protected (in PLS, covered by MAC)   U = unprotected (passes untouched, not covered)
```

A legacy receiver sees standard words. It does not subscribe to `L_S`, so it ignores the seal and behaves exactly as before. An A429X receiver on the same bus keeps a copy of the epoch, verifies the seal, and passes words on according to its delivery mode.

The design pattern of truncated MAC plus truncated freshness counter is proven in automotive SecOC [R10]. SLP adapts it to the constraints of ARINC 429: simplex, broadcast to ≤ 20 receivers, no back channel, 23 usable bits per word, and a fixed 2.8 kwords/s ceiling.

### 5.2 Threat model

| Threat | Example | SLP outcome |
|---|---|---|
| T1 Injection | Tap or rogue device sends a fake word 0o203 (altitude) in a gap | Epoch count or tag mismatch → FAIL. Fingerprint IDS flags the word (§6.6). |
| T2 Modification | Inline device alters data bits | Tag mismatch → FAIL_TAG |
| T3 Replay | Recorded legitimate epochs re-sent later | FV ≤ last accepted → FAIL_REPLAY |
| T4 Deletion / reorder | Words or seals dropped or reordered | FAIL_COUNT / FAIL_TAG / FAIL_TIMEOUT |
| T5 Seal stripping | Attacker removes seals so data passes as if unprotected | Receiver expects seals for the PLS → FAIL_TIMEOUT; Enforce mode withholds the data |
| T6 Masquerade with a legacy LRU | Legacy (unsealing) LRU substituted | Same as T5. LHM fingerprint change. |
| T7 Cross-bus splicing | Seal copied from bus A to bus B | BUS_ID and per-bus keys in the MAC → FAIL_TAG |

Out of scope: denial of service by physically jamming the line (detected by LHM, but it cannot be prevented by protocol), side-channel extraction of keys from the LRU (implementation hardening is an option, §19), and a compromised legitimate transmitter.

### 5.3 Terms

| Term | Definition |
|---|---|
| PLS | Protected Label Set: set of (label, SDI) for one seal group. Tx and Rx must hold identical PLS (from the SLP profile, Appendix B). |
| Seal group | Independent epoch stream on one Tx channel, with its own `L_S`, PLS, N, timers and FV. Up to 4 per channel. |
| Epoch | The ordered sequence of protected words of a group between two seals. 1 ≤ n ≤ 63. |
| Seal | Contiguous burst: Seal Header Word (SHW) + optional Sync Words (SYW) + Tag Words (STW). |
| FV | Freshness Value: 48-bit counter per (channel, group). Every seal (data or sync) uses up one value. |
| TAGW | Number of tag words: 1, 2 or 3. Tag length T = 21·TAGW bits. |
| BUS_ID | 16-bit identifier of the physical bus, assigned in the ICD. |

### 5.4 Seal word formats

All seal words are standard ARINC 429 words on label `L_S`, with correct odd parity. **Payload P21 = ARINC bits 11–31 (21 bits).** The SDI bits (9–10) give the position inside the seal.

**Seal Header Word (SHW), SDI = 00:**

| ARINC bits | Field | Description |
|---|---|---|
| 11–12 | KID | Key slot ID 0–3 |
| 13–18 | CNT | 1–63 = data seal covering CNT words; 0 = SYNC seal |
| 19–31 | FVL / SYNC_INFO | Data seal: FV[12:0]. Sync seal: bits 19–20 = SYNC_TYPE (00 FV only, 01 FV + TIME); 21–31 = 0 |

**Subsequent seal words:** SDI = 01, 10, 11, 01, 10, … in sequence. A receiver shall check the SDI sequence (`[SLP-041]`).

- Data seal: `SHW, STW1 … STW_TAGW`. Tag bits are packed MSB first: STW1 carries tag[T−1 : T−21].
- Sync seal: `SHW, SYW1 … SYWk, STW1 … STW_TAGW`. The sync payload is `FV[47:0]` (k = 3 words), or `FV[47:0] ‖ TIME[47:0]` (k = 5 words). TIME is the transmitter timebase in units of 1.024 µs (`TIME = now_ns[57:10]`), sampled when the seal is computed (≤ 5 µs before the SHW is sent at 100 MHz). The payload is packed MSB first in 21-bit chunks and zero-padded. Payload values are integers whose bit 0 is ARINC bit 11.

**Total seal length:** data seal 2–4 words; sync seal 5–9 words.

### 5.5 MAC construction

- **[SLP-001]** The tag shall be `T` MSBs of `AES-CMAC(K, M)`, where `K` is the 128-bit key selected by (channel, group, KID) and `M` is the byte string below (multi-byte fields big-endian):

The header is exactly one AES block (16 bytes), so the body words are block-aligned:

| Field | Size (bytes) | Value |
|---|---|---|
| DOMAIN | 4 | ASCII `"A4SL"` (0x4134534C) |
| VER_TYPE | 1 | bits [7:4] VERSION = 1; bits [3:0] TYPE: 0 data seal, 1 sync seal |
| BUS_ID | 2 | from profile |
| SEAL_LABEL | 1 | `L_S`, natural octal value |
| GKT | 1 | bits [7:6] GROUP, [5:4] KID, [3:2] TAGW, [1:0] 0 |
| FV | 6 | full 48-bit freshness value of this seal |
| CNT | 1 | n (data) or 0 (sync) |
| BODY | 4·n (data); 8 or 16 (sync) | Data: each protected word as a 32-bit integer (ARINC bit k at integer bit k−1, parity included), in transmission order. Sync: the 32-bit words `{0x0000, FV[47:32]}`, `FV[31:0]` and, when `SYNC_TYPE = 01`, `{0x0000, TIME[47:32]}`, `TIME[31:0]`. |

- **[SLP-002]** The longest data epoch is 16 + 252 = 268 bytes (17 AES blocks). The crypto engine shall finish the longest CMAC in ≤ 600 clk. The implementation fetches the next block's words while AES processes the current block, at about 22 clk per block plus the subkey. Measured: 390 clk for 17 blocks. The engine is shared and its worst-case queue across all channels is bounded (§13.3).
- **[SLP-003]** Tag comparison shall cover all T bits with no data-dependent early exit.
- **[SLP-004]** Build option `G_AUTH_ALG = ASCON`: the tag is the truncated Ascon-AEAD128 tag with empty plaintext, `M` as associated data, and nonce = `DOMAIN ‖ BUS_ID ‖ GROUP ‖ KID ‖ FV ‖ 0-pad` (unique because FV is never reused per key). This option is not FIPS-approved and is offered for low-area devices.

### 5.6 Keys

- **[KEY-001]** The key store shall hold 16 × 128-bit keys. A per-channel/group mapping table maps `(channel, dir, group, KID) → key index`.
- **[KEY-002]** Keys shall be write-only. No register path, debug path or AXIS path shall read back a key or any intermediate value that depends on a key.
- **[KEY-003]** Key-writing transactions shall be accepted only from AXI secure transactions (`AWPROT[1] = 0`) when `G_SECURE_BUS = 1` (the default). A non-secure master is handled as follows:
  - `KEY_DATA` writes have no effect.
  - A key command is rejected, with SLVERR when `GCTRL.STRICT = 1`.
  - The rejection is logged as event 0x08 and raises `GIRQ_MISC.NONSEC_KEY`.

  The ZCU106 evaluation design sets `G_SECURE_BUS = 0` so that a non-secure Linux host can load keys.
- **[KEY-004]** Optional loading paths: (a) plaintext over the secure AXI path; (b) AES Key Wrap (SP 800-38F) under a Key-Encryption-Key held in slot 15; (c) derivation per NIST SP 800-108r1, counter mode, PRF = AES-CMAC, r = 32, L = 128, one iteration: `K_bus = CMAC(K_master, [1]_32 ‖ "A429X-SLPv1" ‖ 0x00 ‖ BUS_ID[15:0] ‖ GROUP[7:0] ‖ KID[7:0] ‖ [128]_32)`. Command: KEY_SEL = destination slot, KEY_DATA0 = {BUS_ID, GROUP, KID}, KEY_CMD = 3 | src_slot << 8. The master key goes to the CMAC engine only, never onto the register bus. The destination becomes valid when the derivation completes, and a zeroize during derivation discards the result. Option (c) gives every bus a unique key from one provisioned master.
- **[KEY-005]** Zeroization is triggered by a `SEC_CTRL.ZEROIZE` write or by asserting `zeroize_n`. The pin is asynchronous: it passes 2 synchronizer clocks and a 4-clk glitch filter, so pulses shorter than 4 clk are ignored. Within 64 clk, zeroization shall clear:
  - all key slots
  - the key staging and derived-key registers
  - the CMAC subkeys, chaining value and tag
  - the AES state and round key

  Measured: ≤ 10 clk after the register write.

  Jobs in flight are cancelled, so no tag is delivered. The core then sets `SEC_STATUS.ZEROIZED` and stops all sealing and verification:
  - Protected words are blocked [SLP-034].
  - Receivers report `UNSYNCED`.

  Recovery is `SEC_CTRL.KAT_RUN`, then key reload, then group re-enable. Outside zeroization the same registers are cleared at the end of every job, so no key-derived value is held while the engine is idle. An embedded assertion checks this.
- **[KEY-006]** Power-up self-test: the AES-CMAC known-answer tests (RFC 4493 vectors) shall run after reset. Key slots stay unusable until the tests pass. A failure sets `SEC_STATUS.KAT_FAIL` and the core stays in the zeroized state.
- **[KEY-007]** Key rollover: the transmitter switches KID only at an epoch boundary. The KID is sampled when a seal is computed, so a KID written while an epoch is open takes effect with that epoch's seal; every epoch is sealed under exactly one KID. Receivers accept any KID whose slot is valid.

> Per-bus keys keep the effect of a compromised receiver within one bus. Because ARINC 429 is simplex, a receiver that holds a group key can still forge only if it is physically wired as a transmitter on that bus. SLP's goal is therefore to authenticate *the wire's legitimate transmitter*, and per-bus symmetric keys are appropriate for that goal.

### 5.7 Freshness value

- **[SLP-010]** The Tx FV per (channel, group) shall be 48 bits, increment by 1 for every seal, and never wrap: the last FV used is 2^48 − 2. When the seal that used it completes, the group stops sealing, raises `FV_EXHAUSTED` and logs event 0x05 with its group. Protected words of a stopped group are then blocked ([SLP-034]). Re-enabling an exhausted group, or an enable whose FV would reach 2^48 − 1, leaves it stopped and raises `FV_EXHAUSTED` / event 0x05 again. A `SYNC_NOW` request on a stopped group is ignored. At the maximum seal rate this is over 10,000 years.
- **[SLP-011]** Persistence: software shall write the starting FV (`TX_FVg`) before enabling the group. On enable the core sets FV = max(written value, last FV used since reset) + 2^`FV_LEAP` (programmable, default 2^20), so the FV never decreases within a power cycle even if software forgets to rewrite it. The core raises `FV_PERSIST_DUE` when the FV of a completed seal is a multiple of 2^`PERSIST_SHIFT` (that is, every 2^`PERSIST_SHIFT` seals; `PERSIST_SHIFT` = 0 disables it) so the host can save the current FV to NVM. With the leap, a value is never reused after an unclean power loss.
- **[SLP-012]** Receiver reconstruction: given the last accepted value `FV_last` and the received 13-bit `r`:
  `c = (FV_last & ~0x1FFF) | r; if (c ≤ FV_last) c += 0x2000;` The seal is accepted only if `c − FV_last ≤ W` (W = 2^`WIN_LOG2`, 1 ≤ WIN_LOG2 ≤ 12, default 10). Otherwise it is rejected as `FAIL_REPLAY`: a replayed (old) FV reconstructs to a far-future value, so the two cases cannot be told apart and are treated the same way. After `K_ALARM` consecutive window rejections the group enters `UNSYNCED` and waits for a valid sync seal (§5.8).
- **[SLP-013]** The receiver `FV_last` initializes from `RX_FVg` (the floor, written by the host from NVM). It updates only after a successful verification.

### 5.8 Sync seals

- **[SLP-020]** The transmitter shall send a sync seal at power-up/enable, every `SYNC_PERIOD` (programmable 0.1–60 s; 0 = only at enable), and on software request. Periodic sync seals are counted on a free-running 100 ms tick, so the first periodic seal after enable comes up to 100 ms early.
- **[SLP-021]** The receiver shall accept a sync seal only if its tag verifies and `FV_sync > FV_last`. It then sets `FV_last = FV_sync` and enters SYNCED.
- **[SLP-022]** With `SYNC_TYPE = 01`, the sync seal carries the transmitter's TIME (48 bits, 1.024 µs units). The receiver records the offset `REMOTE_TIME_OFS = TIME_remote − TS(seal)`. With it, receivers can compute **source-referenced data age**. The comparison is taken modulo 2^58 ns (the range of TIME, about 9.1 years) and sign-extended, so it is valid for absolute (PTP/TAI) time as well as for time since power-up. v1.0 RTL exposes the offset only, not TIME_REMOTE and TIME_LOCAL separately.

Software sets `TIME_CHECK = 1` once the receiver's local time is trusted, for example after a PTP or PPS lock. A sync seal whose TIME differs from local time by more than `TIME_TOL` shall then be rejected, which bounds post-reboot replay (§5.13). The rejection is reported as FAIL_REPLAY and counts toward K_ALARM.

### 5.9 Transmitter behavior

- **[SLP-030]** For each enabled group, the sealer shall classify each word that wins arbitration. Words whose (label, SDI) is in the group PLS are appended to the running epoch and fed to the CMAC (MAC input is buffered: up to 63 words per group).
- **[SLP-031]** The epoch closes when n = `N` (1–63), or `T_EPOCH` (1–1000 ms, counted on the 1 ms tick, so the epoch closes between T_EPOCH − 1 ms and T_EPOCH after its first word) has elapsed since its first word, or software flushes it. The seal shall then be the next transmission on the channel, contiguous and not interleaved with other words.
- **[SLP-032]** Only protected and unprotected traffic words may appear inside an epoch. Seals of different groups shall not interleave.
- **[SLP-033]** The seal shall be ready before the gap after the last protected word ends, so that no extra idle time is inserted. This holds at HS and LS for every configuration (§13.3). At faster custom rates the bound holds while the CMAC jobs that can be pending at once (390 clk each) fit in one word time: at 1 Mbps and 100 MHz (3600 clk) that is up to 9 channels sealing in the same word time.
- **[SLP-034]** A protected word shall be transmitted only if the transmitter can seal it at that moment. That requires all of the following:
  - sealing enabled
  - group enabled and not stopped
  - key slot valid
  - FV not exhausted
  - core not zeroized

  Otherwise the word is dropped, counted in `TX_CNT_BLOCKED` and raises `TX_IRQ.SEAL_BLOCKED`. If the key or security is lost while an epoch is open, the epoch is abandoned (event 0x05). The group stays stopped until it is re-enabled, and receivers time out the words already sent. `TX_FIFO_CTL.BYPASS_SEAL` sends a protected word unsealed only when `TEST_MODE = 1` (negative tests). With `TX_CTRL.SEAL_EN = 0` the channel is a legacy transmitter and the protect map is ignored.
- **[SLP-035]** Counters per group: epochs sealed, sync seals, current FV (`TX_FVg` read while enabled returns the next FV to be used).

### 5.10 Receiver behavior

Receiver modes per channel (`RX_CTRL.SEC_MODE`):

| Mode | Data delivery | Use |
|---|---|---|
| OFF | Legacy. Seal label treated as an ordinary label. | Legacy bus |
| MONITOR | Words delivered at once (AUTH = PENDING), mailboxes updated at once. Verdicts are only logged and counted. | Deployment step 1: "observe before enforce" |
| EARLY | Like MONITOR, plus a VERDICT record (§4.7) after each seal. Software can roll back or downgrade affected data. | Latency-critical consumers |
| ENFORCE | Protected words held in quarantine until the seal verifies. PASS releases them with AUTH = PASS and updates mailboxes. FAIL discards them (or stores them with failure status if `STORE_ERR = 1`) and leaves mailboxes untouched. | Target operational mode |

- **[SLP-040]** Each group's quarantine shall hold 63 words plus metadata. If the 64th protected word arrives without a seal, the core shall raise FAIL_COUNT for the buffer and restart.
- **[SLP-041]** When the SHW arrives, the receiver shall check, in order: SDI sequence and length; KID slot valid; CNT == number of quarantined protected words (else FAIL_COUNT); FV reconstruction and window (else FAIL_REPLAY/UNSYNCED); and tag (else FAIL_TAG).
  - An unusable KID gives UNSYNCED.
  - A malformed seal fails the epoch as FAIL_TAG. Malformed means SDI out of sequence, or a protected word arriving inside a seal.
  - A protected word that interrupts a seal opens the next epoch. This lets the receiver recover when a real seal word is lost on the line.
- **[SLP-042]** If `T_SEAL_MAX` passes after the first quarantined word with no seal, the result is FAIL_TIMEOUT (covers stripping, T5). The recommended value is 2 × T_EPOCH + 4 word times. The driver or profile tool programs it. The register resets to 0 (off) because the receiver does not know T_EPOCH.
- **[SLP-043]** Verdict record (EARLY mode, or any mode with `VERDICT_EN`), flagged with `META.VERDICT = 1` and `META.AUTH = RESULT`:

| DATA bits | Field |
|---|---|
| [31:16] | SEQ_FIRST |
| [15:10] | CNT (words held) |
| [9:7] | RESULT (AUTH code) |
| [6:5] | GROUP |
| [4:3] | KID |
| [2:0] | 0 |

- **[SLP-044]** Alarm: after `K_ALARM` consecutive failures (default 3), or `K_RATE` failures in a sliding 1 s window (K_RATE: roadmap, not in v1.0 RTL), the receiver shall raise `SEC_ALARM` and latch an event-log entry.
- **What counts:** failed data seals and rejected sync seals. UNSYNCED results (no usable key) do not count.
- **ALARM_POLICY = 1:** in ENFORCE mode, data for that group stays withheld until a valid sync seal resynchronizes it.
  - A withheld epoch whose tag verifies is reported as UNSYNCED and counted in AUTH_FAIL_CNT.
  - Its FV is still accepted (FV_last advances), so the epoch cannot be replayed later.
- **[SLP-045]** Unprotected words shall be delivered as in legacy operation (AUTH = NONE), whatever the mode.
- **[SLP-046]** Words with parity or frame errors that carry a protected label shall still count toward CNT. That way a corrupted word fails the epoch (integrity) and does not shift alignment.

Receiver state per group: `DISABLED → UNSYNCED → SYNCED ⇄ ALARM`. UNSYNCED leaves for SYNCED on a valid sync seal, or on a valid data seal within the window of the floor.

### 5.11 Overhead, latency and strength (design guide)

**Bus overhead** = S / (N + S), where S = 1 + TAGW (data seal). Sync seals add (5–9 words)/SYNC_PERIOD, which is negligible.

| N | TAGW=1 (21-bit) | TAGW=2 (42-bit) | TAGW=3 (63-bit) |
|---|---|---|---|
| 8 | 20.0 % | 27.3 % | 33.3 % |
| 16 | 11.1 % | 15.8 % | 20.0 % |
| 32 | 5.9 % | 8.6 % | 11.1 % |
| 63 | 3.1 % | 4.5 % | 6.0 % |

**Added latency (ENFORCE)** ≤ min(T_EPOCH, time to collect N protected words) + S × word time + verification (≤ 3 µs). At HS, word time is 360 µs. EARLY mode adds no latency.

**Forgery resistance:** online attempts per second are capped by the bus at about `2777 / (2 + TAGW)`. Each failure is also reported, and K_ALARM failures trigger an alarm.

| Tag | Mean time to one successful blind forgery at maximum attempt rate | Recommendation |
|---|---|---|
| 21-bit | ≈ 40 minutes (after ~2 million reported verification failures) | Integrity and anomaly detection only |
| 42-bit | ≈ 200 years | **Default** for operational data |
| 63-bit | > 10^8 years | High assurance, commands, DAL A-critical security |

Integrity bonus: any random corruption of a protected word is detected with probability 1 − 2^−T, far above the single parity bit.

### 5.12 Backward-compatibility rules (integrator)

- **[SLP-050]** `L_S` shall be a label that no receiver on the bus uses (from the ICD). The IP never hard-codes `L_S`.
- **[SLP-051]** The integrator shall confirm that bus load after sealing stays within the program's limit. `RX_LOAD` and the A429X profile tool report it (Appendix B).
- **[SLP-052]** Legacy receivers that log unknown labels as faults (rare) shall be identified during interoperability testing (§16.6 ZT-30).
- **[SLP-053]** A sealing transmitter can coexist with non-A429X receivers. An A429X receiver in OFF mode coexists with a legacy transmitter.

### 5.13 Residual risks (stated honestly)

1. **Replay after a receiver reboot.** A simplex bus has no challenge-response path. After a reboot the receiver trusts its persisted floor. Recent recordings newer than that floor can be replayed until the next legitimate seal pushes FV beyond them. Mitigations: persist often (`FV_PERSIST_DUE`), and use time-checked sync seals (`[SLP-022]`) when the receiver has a trusted time source. The replayed data is still authentic recent data, and the refresh monitor bounds how stale it can be.
2. **Key compromise of the transmitting LRU** defeats SLP for that bus. Per-bus keys contain the damage.
3. **Seal-label collision** if the ICD is wrong. This is mitigated by the profile tool and interop testing.

---

## 6. Line Health Monitor (LHM) and Physical Fingerprint IDS

### 6.1 Concept

Standard ARINC 429 receivers turn an analog waveform into two logic levels and throw away everything else. A429X measures *how* each word arrived. It builds per-link statistics and trends, so maintenance can see a link degrading weeks before it fails. The same measurements identify *who* drove the line.

- **[LHM-003]** One LHM engine shall serve all Rx channels. Each channel hands over its per-word measurements, and the engine serves pending channels round-robin (about 3 µs per word at 100 MHz). Per-channel statistics, limits and fingerprint baselines are kept in RAM indexed by channel. Words arrive at most once per 36 bit times per channel, so a single engine sustains 16 channels at full HS load. The register map is per channel, as if each channel had its own monitor.

### 6.2 Front-end tiers

| Tier | External hardware | Inputs to IP | Metrics available |
|---|---|---|---|
| **FE0 Standard** | Any standard ARINC 429 line receiver | `rx_hi`, `rx_lo` | Bit period / rate offset (ppm), pulse widths, duty asymmetry, jitter, gaps, glitches, invalid states |
| **FE1 Margin** | Line receiver plus 2 extra comparators at a programmable *margin* threshold (for example ±8.0 V through a DAC) | + `rx_hi_m`, `rx_lo_m` | FE0 + slew time between decision and margin thresholds (rise and fall per polarity), margin-miss rate (amplitude degradation), plateau width |
| **FE2 Analog** | Attenuator plus ADC per leg (≥ 10 MSPS, ≥ 10 bit) | `s_axis_adc` (A and B leg samples) | FE1 + absolute HI/LO amplitude, 10–90 % rise/fall, overshoot, ringing, NULL noise RMS, common-mode, eye height/width |

- **[LHM-001]** The tier shall be selected at build time (`G_LHM_TIER`) and at run time (`RX_CTRL.FE_MODE` ≤ build tier). FE_MODE reads back as written; the effective mode is min(FE_MODE, build tier).
- **[LHM-002]** In FE2, a digital comparator with programmable thresholds and hysteresis shall derive `rx_hi` and `rx_lo` from the samples. Products that need certified decoding should keep a standard comparator receiver in parallel. The IP supports FE2 used only as a monitor (`FE2_MONITOR_ONLY`).

### 6.3 Metrics

- **[LHM-010]** The following metrics shall be measured per received word (time metrics at 1 clk resolution, reported in clk/16 fixed point after averaging):

| ID | Metric | Tier |
|---|---|---|
| 0 | BIT_PERIOD (mean over the word) | FE0 |
| 1 | PW_HI (pulse width of 1-bits) | FE0 |
| 2 | PW_LO (pulse width of 0-bits) | FE0 |
| 3 | GAP_TIME (preceding gap) | FE0 |
| 4 | JITTER (peak-to-peak variation of the bit period within the word) | FE0 |
| 5 | RISE_HI (decision→margin, HI) | FE1 |
| 6 | FALL_HI (margin→decision, HI) | FE1 |
| 7 | RISE_LO | FE1 |
| 8 | FALL_LO | FE1 |
| 9 | AMP_HI (plateau mean, mV) | FE2 |
| 10 | AMP_LO | FE2 |
| 11 | NULL_NOISE_RMS (mV) | FE2 |
| 12 | COMMON_MODE (mV) | FE2 |
| 13 | OVERSHOOT (%) | FE2 |
| 14 | MARGIN_MISS (bits per word below margin) | FE1 |
| 15 | GLITCHES (per word) | FE0 |

- **[LHM-011]** Rate offset in ppm relative to the nominal rate, (f − f_nom)/f_nom (positive when the transmitter is fast), shall be derived from an EMA (α = 2^−EMA_K) of the unquantized word span, the source of metric 0, and reported as `RATE_PPM` (signed, saturating at ±32767 ppm). With the default EMA_K = 4 (16 words), resolution is ≤ 10 ppm at HS with a 100 MHz clock.

### 6.4 Statistics engine

- **[LHM-020]** For each metric: last, min, max (resettable), EMA mean and EMA variance (α = 2^−k, k = 2–12 programmable), and sample count.
- **[LHM-021]** Event counters: parity, frame, gap, rate, glitch, invalid-state, margin-miss words.
- **[LHM-022]** Trend buffer: every `TREND_PERIOD` (1 s–1 h) the core shall snapshot the mean, min and max of all enabled metrics into a 64-entry ring per channel (BRAM), or stream it to AXIS. Long-term trends can then be read out after a flight.
- **[LHM-023]** Metric access is indirect: write `LHM_SEL` (metric ID), then read `LHM_LAST/MIN/MAX/MEAN/VAR/CNT`. All are latched coherently at the `LHM_SEL` write. MEAN returns the integer part of the fixed-point (q8) EMA. VAR is the EMA of the squared deviation (deviation clamped to 16 bits).

### 6.5 Health status and score

- **[LHM-030]** Each metric shall have programmable two-sided WARN and ALARM limits. The status per metric is OK, WARN or ALARM, evaluated on the EMA mean (and, for MARGIN_MISS/GLITCHES, on the rate). The WARN bitmap also includes metrics that are in ALARM.
- **[LHM-031]** Health score `HS` (0–255) = 255 − Σ_i (WARN_i ? w_i : 0) − Σ_i (ALARM_i ? 4·w_i : 0), saturating at 0. Weights w_i are programmable (4 bits). A metric in ALARM is charged 4·w_i only. `LHM_WARN` / `LHM_ALARM` interrupts are raised when a metric enters WARN or ALARM, not when it returns to OK.
- **[LHM-032]** Default limits shall ship as SLP-profile-style presets for HS and LS buses (driver datasheet nominal ± tolerance). They are refined by field data (roadmap).

> **Value example:** a chafed shield or corroded pin raises series resistance and capacitance. Slew times rise and margin misses appear (FE1) long before the signal falls below the ±6.5 V decision threshold and parity errors start. LHM reports WARN while data is still 100 % correct, so the fix becomes a planned maintenance action instead of an AOG event or an NFF removal.

### 6.6 Physical Fingerprint IDS

- **[FP-001]** The feature vector per word shall be F = {SPAN, PW_HI, PW_LO, RISE_HI, FALL_HI, RISE_LO, FALL_LO}, limited to what the active tier provides (FE0: the first three). SPAN is the 31-interval word span in clk, the unaveraged source of BIT_PERIOD; AMP_HI/AMP_LO are reserved for FE2. **With FE0 only, the transmitter's oscillator offset and duty asymmetry already identify a source.** Per-word resolution is set by the clock: at HS with 100 MHz, one span clock is 32 ppm. Measured (HS, 100 MHz, L = 8, FE0, 400 legitimate + 100 rogue words per case): ±200 ppm detected on 62–65 % of words (varies run to run with the NCO phase) at the default `D_WORD_TH` = 6 and on 100 % at `D_WORD_TH` = 5, with 0/400 false positives in both cases; ±20 and ±50 ppm are below the per-word resolution and are not detected by FE0. A false-positive rate below 10^-4 has not yet been demonstrated (it needs ≥ 3·10^4 legitimate words).
- **[FP-002]** LEARN mode: over 2^L words (L = 4–15), the core computes μ_i. Over the next 2^L words it computes the scale s_i as the **mean absolute deviation** (MAD, 4 fractional bits; cheaper than σ and needs no square root). It then stores μ_i and `INV_i = 2^20 / max(MAD_q4, 16)`, so the floor is one feature unit. When learning completes the mode switches to DETECT automatically. The baseline (μ, INV) can be read and written by software, so it can be learned once (for example at installation) and restored at boot. `FP_MU` reads and writes the integer part of μ (the fractional bits used by [FP-005] are cleared on write), and MAD is computed against the integer μ. `LEARN_DONE` is set only by hardware learning, so it reads 0 after a software restore. Reset does not clear the baseline RAM: a restore writes every feature.
- **[FP-003]** DETECT mode: per word, `D = Σ_i min((|x_i − μ_i| · INV_i) >> 16, 15)` (in MAD units), using one time-multiplexed multiplier. `D > D_WORD_TH` sets `META.FP_ANOMALY` on that word. An EMA of D (α = 1/8, 4 fractional bits) above `D_WIN_TH` raises `FP_ALARM` (sustained change: LRU swap, rogue takeover or degradation). `FP_STATUS.ALARM` follows (D_ema > D_WIN_TH) and is not sticky; the `FP_ALARM` interrupt and the 0x04 event-log entry are raised on entry only.
- **[FP-004]** Per-word anomaly flags shall be available in all SLP modes. A policy option (`FP_POLICY`) can treat FP_ANOMALY on a protected word as an SLP failure. This defense-in-depth catches gap injection even before the seal arrives. With `FP_POLICY = 1` the flagged word is handled as follows:
- It is rejected at once with AUTH = FAIL_TAG and stored only if `STORE_ERR = 1`. This applies in every mode.
- It is excluded from the epoch, so an authentic epoch around it still verifies.
- It is counted in `FP_ANOM_CNT`, not in AUTH_FAIL_CNT, and logs no event.

A sustained anomaly is reported by FP_ALARM (event 0x04).
- **[FP-005]** Slow drift adaptation: an optional slow EMA update of μ (`FP_CTRL.ADAPT`, α = 2^−ADAPT_K, ADAPT_K 1–15, 0 = 2^−16) for temperature and aging. μ is kept with 16 fractional bits. It is frozen while FP_ALARM is active.

### 6.7 Tx Readback Monitor

- **[RB-001]** With `G_RB_EN = 1`, each Tx channel shall have a readback receiver input (`rb_hi`, `rb_lo`, and optionally margin and ADC) connected to its own line on the PCB (wrap-around).
- **[RB-002]** Compare: each transmitted word shall be decoded from readback and compared with what was sent. Mismatch → `RB_MISMATCH`. No readback within 1 word time → `RB_LOST` (open driver or short).
- **[RB-003]** Foreign activity: any qualified pulse on readback while the channel's own encoder is in NULL or gap → `RB_FOREIGN`. This detects contention and a second transmitter on a simplex bus (G6) at the source.
- **[RB-004]** LHM-RB: the LHM metrics of §6.3 apply to the readback path, giving driver health and load signature. With FE2, a change in amplitude or ringing indicates a change in bus loading (new tap, shorted receiver, broken shield). Load-signature detection is marked *experimental* in v1.0.

---

## 7. Timebase and time distribution

- **[TIME-001]** A 64-bit timebase in nanoseconds shall advance by a programmable fixed-point increment per clk (`TIME_INC`, 8.24 format), so its rate is exact at any core clock.
- **[TIME-002]** Sync sources: software load (`TIME_SET`), rising edge of `pps_in` (TIME takes the value `TIME_PPS_VAL` at the pin edge, with the 2-clk synchronizer latency compensated; the core then adds 1 s to `TIME_PPS_VAL` for the next pulse, and captures the phase error TIME − `TIME_PPS_VAL` at the edge in `TIME_PPS_ERR` to ±1 clk), or an external 64-bit time bus (`time_in`, for example from a PTP/1588 core in the PS or in fabric).
- **[TIME-003]** Reading `TIME_LO` shall latch `TIME_HI` atomically.
- **[TIME-004]** Data age for any FIFO or mailbox entry = `TIME − TS`. When SLP time-sync is in use, the receiver shall also expose the remote-to-local offset per group (`[SLP-022]`).

---

## 8. External interfaces and parameters

### 8.1 Build-time parameters (generics)

| Parameter | Range | Default | Description |
|---|---|---|---|
| `G_CLK_HZ` | 20e6–250e6 | 100e6 | Core clock frequency (for presets and timing defaults) |
| `G_NUM_TX` | 1–16 | 4 | Transmit channels |
| `G_NUM_RX` | 1–16 | 4 | Receive channels |
| `G_TX_FIFO_DEPTH` | 16–1024 | 64 | Words |
| `G_RX_FIFO_DEPTH` | 16–4096 | 256 | Entries (128 bit) |
| `G_SCHED_ENTRIES` | 0, 64, 256 | 64 | Scheduler table size |
| `G_MAILBOX` | 0, 1, 2 | 1 | None / 256 / 1024 entries |
| `G_AXIS_EN` | 0/1 | 0 | AXI4-Stream data ports |
| `G_AUTH_EN` | 0/1 | 1 | SLP and key store |
| `G_AUTH_ALG` | CMAC, CMAC+ASCON | CMAC | Crypto algorithms |
| `G_SECURE_BUS` | 0/1 | 1 | Enforce AxPROT for key access |
| `G_LHM_TIER` | 0–3 | 2 | 0 none, 1 FE0, 2 FE1, 3 FE2 |
| `G_FP_EN` | 0/1 | 1 | Fingerprint IDS |
| `G_RB_EN` | 0/1 | 1 | Tx readback |
| `G_ECC_EN` | 0/1 | 1 | SECDED on BRAM (label table, mailbox, FIFOs) |
| `G_CFG_PROT` | 0/1/2 | 1 | Config register protection: none / parity scrub / TMR |
| `G_VLE_EN` | 0/1 | 0 | Include Virtual Line Emulator (Lab only, §16.3) |

### 8.2 Port list (top level `a429x_top`)

| Port | Dir | Width | Description |
|---|---|---|---|
| `aclk` | in | 1 | Core clock |
| `aresetn` | in | 1 | Active-low reset, synchronized internally |
| `s_axil_*` | — | AXI4-Lite | 32-bit data, 19-bit address (512 KB), AWPROT/ARPROT honored |
| `m_axis_rx_*` | out | 128 + TUSER[7:0] | Rx entries (TUSER = channel ID, entry type) |
| `s_axis_tx_*` | in | 64 + TDEST[3:0] | `{CTL[31:0], WORD[31:0]}` to channel TDEST |
| `irq` | out | 1 | Level-high combined interrupt |
| `irq_vec` | out | NUM_TX+NUM_RX+1 | Optional per-channel interrupts |
| `pps_in` | in | 1 | Pulse-per-second |
| `time_in`, `time_in_valid` | in | 64, 1 | External timebase (optional) |
| `zeroize_n` | in | 1 | Async zeroize request |
| `tx_hi[n]`, `tx_lo[n]` | out | NUM_TX | RZ drive to line driver |
| `tx_slope[n]`, `tx_en[n]` | out | NUM_TX | Slew select, driver enable |
| `rb_hi[n]`, `rb_lo[n]`, `rb_hi_m[n]`, `rb_lo_m[n]` | in | NUM_TX | Readback (margin optional) |
| `rx_hi[n]`, `rx_lo[n]` | in | NUM_RX | Line receiver outputs |
| `rx_hi_m[n]`, `rx_lo_m[n]` | in | NUM_RX | FE1 margin comparators |
| `s_axis_adc_*[n]` | in | 32 (A[15:0], B[15:0]) | FE2 samples, any rate ≤ aclk |
| `dbg_*` | out | — | Debug bus (decoder state, NCO, verdicts) for ILA |

- **[IF-001]** All asynchronous inputs (`rx_*`, `rb_*`, `pps_in`, `zeroize_n`) shall be synchronized with ≥ 2 flip-flops (marked ASYNC_REG) inside the IP.
- **[IF-002]** The AXI4-Lite slave shall support back-to-back transactions (all registers are 32-bit: `WSTRB` is ignored and every write updates the whole register), return SLVERR for writes to read-only or locked registers when `GCTRL.STRICT = 1`, and return DECERR for unmapped addresses. In v1.0 RTL, SLVERR is generated by the core register block (global, timebase, interrupt and security registers, including rejected key commands). Channel registers ignore writes to read-only offsets without an error response.

---

## 9. Register map

All registers are 32 bits. RW = read/write, RO = read-only, W1C = write 1 to clear, RO/W1C on a counter = cleared by any write (the value is ignored), WO = write-only, SC = self-clearing. Total address space is 512 KB (address bits [18:0]).

### 9.1 Address regions

| Base | Size | Region |
|---|---|---|
| 0x0_0000 | 0x100 | Global |
| 0x0_0100 | 0x100 | Timebase |
| 0x0_0200 | 0x200 | Interrupt summary and event log |
| 0x0_0400 | 0x400 | Security (key store, crypto, mapping) |
| 0x0_1000 | 16 × 0x100 | Tx channel *n* (0x1000 + n·0x100) |
| 0x0_2000 | 16 × 0x100 | Rx channel *n* |
| 0x0_3000 | 16 × 0x100 | LHM / FP for Rx *n* |
| 0x0_4000 | 16 × 0x100 | LHM-RB for Tx *n* |
| 0x0_5000 | 0x1000 | VLE / Lab (if `G_VLE_EN`) |
| 0x0_6000 | 16 × 0x200 | Tx protect map *n*: 1024 entries × 4 bits {rsvd, GRP[1:0], PROT}, 8 per word, index `{SDI, LABEL}` |
| 0x0_8000 | 16 × 0x800 | Tx scheduler table *n* (256 × 8 B) |
| 0x1_0000 | 16 × 0x1000 | Rx label table *n* (1024 × 4 B) |
| 0x2_0000 | 16 × 0x1000 | LHM trend ring *n* (64 entries) |
| 0x4_0000 | 16 × 0x4000 | Rx mailbox *n* (1024 × 16 B) |

### 9.2 Global (0x0_0000)

| Off | Name | Acc | Description |
|---|---|---|---|
| 0x00 | ID | RO | 0x429A_0001 |
| 0x04 | VERSION | RO | [31:24] major, [23:16] minor, [15:0] build |
| 0x08 | CAP0 | RO | [4:0] NUM_TX, [12:8] NUM_RX, [16] AUTH, [17] ASCON, [19:18] LHM_TIER, [20] FP, [21] RB, [22] AXIS, [23] ECC, [25:24] MAILBOX, [26] VLE, [29:27] NUM_GROUPS |
| 0x0C | CAP1 | RO | [3:0] log2 TX_FIFO, [7:4] log2 RX_FIFO, [9:8] SCHED (0/64/256 enc.), [31:16] CLK_MHZ |
| 0x10 | GCTRL | RW | [0] EN, [1] SRST (SC), [2] LABEL_NAT (default 1), [3] STRICT, [4] LOCK (config lock until reset; v1.0 RTL locks GCTRL itself, and the full configuration lock is v1.1), [8] FREEZE_ALL |
| 0x14 | GSTATUS | RO | [0] READY, [1] ECC_CE, [2] ECC_UE, [3] CFG_SEU, [4] LOCKED |
| 0x18 | SCRATCH | RW | |
| 0x20 | ECC_CE_CNT | RO/W1C | Correctable errors |
| 0x24 | ECC_UE_ADDR | RO | Last uncorrectable address |

### 9.3 Timebase (0x0_0100)

| Off | Name | Acc | Description |
|---|---|---|---|
| 0x00 | TIME_LO | RO | Reading latches TIME_HI |
| 0x04 | TIME_HI | RO | |
| 0x08 | TIME_SET_LO | RW | |
| 0x0C | TIME_SET_HI | RW | |
| 0x10 | TIME_CTRL | RW | [0] LOAD (SC), [2:1] SRC 00 free/01 PPS/10 ext, [3] PPS_EDGE |
| 0x14 | TIME_INC | RW | ns per clk, 8.24 fixed point (100 MHz → 0x0A00_0000) |
| 0x18 | TIME_PPS_ERR | RO | Signed ns error at last PPS |
| 0x1C | SCHED_TICK | RW | Global scheduler tick in µs (default 1000) |
| 0x20 | RATE_TICK | RW | Refresh monitor tick in µs (default 1000) |
| 0x24 | TIME_PPS_VAL_LO | RW | TIME loaded at the next PPS edge (default 1 s); advanced by 1 s at each PPS [TIME-002] |
| 0x28 | TIME_PPS_VAL_HI | RW | |

### 9.4 Interrupt summary and event log (0x0_0200)

| Off | Name | Acc | Description |
|---|---|---|---|
| 0x00 | GIRQ_RX | RO | [15:0] Rx channel pending summary |
| 0x04 | GIRQ_TX | RO | [15:0] Tx channel pending summary |
| 0x08 | GIRQ_MISC | W1C | [0] SEC, [1] TIME_PPS, [2] ECC_UE, [3] CFG_SEU, [4] ZEROIZED, [5] KAT_FAIL, [6] EVLOG_NE, [7] NONSEC_KEY (rejected non-secure key access) |
| 0x0C | GIRQ_EN | RW | [0] global enable, [1] misc enable |
| 0x10 | EVLOG_STATUS | RO | [7:0] count, [8] overflow |
| 0x14 | EVLOG_POP | RO | Reading pops the oldest entry and returns its info word (0 when empty); latches the entry into EVLOG_D0..D2 |
| 0x18–0x20 | EVLOG_D0..D2 | RO | D0 = info {type[31:24], channel[23:20], group[19:18], 0, code[10:8], count[7:0]}; D1/D2 = timestamp lo/hi (ns). Types: 0x01 auth fail, 0x02 alarm, 0x03 foreign transmitter, 0x04 FP alarm, 0x05 seal exhausted / no key, 0x06 zeroize, 0x07 KAT fail, 0x08 non-secure key access |

### 9.5 Security (0x0_0400)

| Off | Name | Acc | Description |
|---|---|---|---|
| 0x00 | SEC_CTRL | RW | [0] EN, [1] ZEROIZE (SC), [2] KAT_RUN (SC), [3] KEY_LOCK (no further key writes until reset) |
| 0x04 | SEC_STATUS | RO | [15:0] slot valid, [16] KAT_PASS, [17] KAT_FAIL, [18] ZEROIZED, [19] BUSY (self-test or derivation), [20] CMD_ERR, [21] KEY_LOCK |
| 0x08 | KEY_SEL | RW | [3:0] slot |
| 0x0C | KEY_CMD | WO | 1 LOAD_PLAIN, 2 UNWRAP (KEK slot 15), 3 DERIVE (from slot in [11:8]), 4 INVALIDATE. v1.0 RTL implements 1, 3 and 4. 2 (UNWRAP) sets SEC_STATUS.CMD_ERR (bit 20), as does any key command while a derivation is in flight |
| 0x10–0x1C | KEY_DATA0..3 | WO | 128-bit key / derivation context |
| 0x20–0x34 | KEY_WRAP0..5 | WO | 192-bit wrapped key (AES-KW) |
| 0x40 | BUS_ID_TX[n] packed | RW | Two 16-bit BUS_IDs per register (0x40–0x5C for 16 Tx) |
| 0x60 | BUS_ID_RX[n] packed | RW | 0x60–0x7C |
| 0x100–0x2FF | KEYMAP | RW | One entry per (dir, channel, group): [15:0] = four 4-bit key indices for KID 0–3. Tx n group g at 0x100 + 4·(4n + g), Rx n group g at 0x200 + 4·(4n + g) |
| 0x300 | CRYPTO_UTIL | RO | [15:0] crypto engine busy cycles (saturating, diagnostic) |

### 9.6 Tx channel n (0x0_1000 + n·0x100)

| Off | Name | Acc | Description |
|---|---|---|---|
| 0x00 | TX_CTRL | RW | [0] EN, [2:1] RATE (00 HS, 01 LS 12.5k, 10 custom), [4:3] PARITY (00 odd, 01 even, 10 transparent), [5] SLOPE_OVR, [6] SLOPE_VAL, [7] LOOP_INT, [8] SCHED_EN, [9] SEAL_EN, [10] RB_EN, [11] FLUSH (SC), [12] HOLD, [23:16] GAP (≥4), [24] TEST_MODE, [25] PRIO_FIFO |
| 0x04 | TX_STATUS | RO | [11:0] FIFO level, [12] EMPTY, [13] FULL, [14] BUSY, [15] HELD |
| 0x08 | TX_NCO_INC | RW | Custom rate increment |
| 0x0C | TX_FIFO_DATA | WO | Push word |
| 0x10 | TX_FIFO_CTL | RW | Applies to next push: [0] TIMED, [4:1] INJECT code, [11:5] INJECT param, [12] BYPASS_SEAL (test), [13] STICKY |
| 0x14 | TX_FIFO_TIME_LO | RW | Release time (with TIMED) |
| 0x18 | TX_FIFO_TIME_HI | RW | |
| 0x1C | TX_FIFO_THRESH | RW | |
| 0x20 | TX_IRQ_STATUS | W1C | [0] EMPTY, [1] THRESH, [2] OVF, [3] SCHED_LATE, [4] RB_MISMATCH, [5] RB_LOST, [6] RB_FOREIGN, [7] SEAL_SENT, [8] FV_PERSIST_DUE, [9] FV_EXHAUSTED (or epoch abandoned: no key), [10] LHM_WARN, [11] LHM_ALARM, [12] BULK_DONE, [13] SEAL_BLOCKED (protected word dropped, [SLP-034]) |
| 0x24 | TX_IRQ_EN | RW | Same bit layout |
| 0x28 | SCHED_CTRL | RW | [8:0] active entries, [9] FREEZE, [10] RESYNC (SC) |
| 0x30–0x3C | TX_CNT_WORDS / SEALS / OVF / LATE | RO/W1C | Counters |
| 0x40 + g·8 | SEAL_G[g]_CFG0 | RW | [0] EN, [8:1] L_S, [14:9] N, [16:15] TAGW, [18:17] KID, [19] SYNC_NOW (SC), [20] FLUSH (SC), [22:21] SYNC_TYPE |
| 0x44 + g·8 | SEAL_G[g]_CFG1 | RW | [9:0] T_EPOCH (ms, 0 = count only), [25:10] SYNC_PERIOD (×0.1 s, 0 = at enable only), [30:26] PERSIST_SHIFT |
| 0x60 + g·8 | TX_FV_G[g]_LO | RW* | Writable when group disabled; then RO = current FV |
| 0x64 + g·8 | TX_FV_G[g]_HI | RW* | [15:0] |
| 0x80 | FV_LEAP | RW | log2 leap (default 20) |
| 0x84–0x90 | SEAL_CNT_G[0..3] | RO | Epochs sealed |
| 0x98 | TX_CNT_BLOCKED | RO/W1C | Protected words dropped because they could not be sealed [SLP-034] |
| 0xB0–0xBC | SYNC_CNT_G[0..3] | RO | Sync seals sent per group [SLP-035] |
| 0xA0 | RB_CTRL | RW | [0] COMPARE_EN, [1] FOREIGN_EN, [15:8] RB_DELAY_MAX (bits) |
| 0xA4 | RB_ERR_CNT | RO/W1C | Readback mismatch + lost |
| 0xA8 | BULK_CTRL | RW | [0] START, [12:1] COUNT, [13] SRC_AXIS (roadmap; reads 0 in v1.0 RTL) |
| 0xAC | RB_FOREIGN_CNT | RO/W1C | Foreign-transmitter detections [RB-003] |

### 9.7 Rx channel n (0x0_2000 + n·0x100)

| Off | Name | Acc | Description |
|---|---|---|---|
| 0x00 | RX_CTRL | RW | [0] EN, [2:1] RATE (00 HS, 01 LS, 10 AUTO, 11 custom), [3] PARITY_CHK, [4] FILTER_EN, [5] SDI_FILT, [6] MBX_EN, [7] STREAM_EN, [9:8] FE_MODE, [12:10] SEC_MODE (0 OFF, 1 MONITOR, 2 EARLY, 3 ENFORCE), [13] STORE_ERR, [14] STORE_SEAL, [15] FLUSH (SC), [23:16] GLITCH_MIN, [27:24] TOL %, [28] WAIT_GAP, [29] VERDICT_EN |
| 0x04 | RX_STATUS | RO | [12:0] FIFO level, [13] EMPTY, [14] FULL, [17:16] RATE_DET (HS/LS/UNK), [18] ACTIVE |
| 0x08 | RX_FIFO_DATA | RO | Pop |
| 0x0C | RX_FIFO_META | RO | Shadow of popped entry |
| 0x10 | RX_FIFO_TS_LO | RO | Shadow |
| 0x14 | RX_FIFO_TS_HI | RO | Shadow |
| 0x18 | RX_FIFO_THRESH | RW | |
| 0x1C | RX_NCO / GAP_DET | RW | [15:0] custom bit period (clk), [23:16] GAP_DET (bits ×0.25; 0 = default 1.75) |
| 0x20 | RX_IRQ_STATUS | W1C | [0] FIFO_NE, [1] THRESH, [2] OVF, [3] PARITY_ERR, [4] FRAME_ERR, [5] RATE_CHANGE, [6] LABEL_MATCH, [7] STALE, [8] TOO_FAST, [9] AUTH_FAIL, [10] SEAL_TIMEOUT, [11] UNSYNCED, [12] SEC_ALARM, [13] FP_ANOMALY, [14] FP_ALARM, [15] LHM_WARN, [16] LHM_ALARM, [17] VERDICT, [18] INVALID_STATE |
| 0x24 | RX_IRQ_EN | RW | |
| 0x28 | RX_RATE_MEAS | RO | Last measured bit period (clk) |
| 0x2C | RX_LOAD | RO | [9:0] load ×0.1 %, [31:16] window ms (RW in CFG) |
| 0x30–0x48 | RX_CNT_WORDS / PARITY / FRAME / GAP / RATE / GLITCH / INVALID | RO/W1C | Counters (write clears) |
| 0x4C | RX_CNT_OVF | RO/W1C | FIFO overflow drops |
| 0x50 | RX_CNT_MARGIN | RO/W1C | Words with at least one pulse that never reached the margin threshold (FE1) [LHM-021] |
| 0x60 + g·8 | RX_SEAL_G[g]_CFG0 | RW | [0] EN, [8:1] L_S, [10:9] TAGW, [14:11] WIN_LOG2, [17:15] K_ALARM, [18] TIME_CHECK, [19] ALARM_POLICY, [20] FP_POLICY |
| 0x64 + g·8 | RX_SEAL_G[g]_CFG1 | RW | [11:0] T_SEAL_MAX (ms, 0 = off), [27:12] TIME_TOL (units of 16.384 µs) |
| 0x80 + g·8 | RX_FV_G[g]_LO | RW* | Floor (write when disabled), else FV_last |
| 0x84 + g·8 | RX_FV_G[g]_HI | RW* | |
| 0xA0 + g·4 | RX_SEC_STATE_G[g] | RO | [1:0] state, [7:4] consecutive fails |
| 0xB0 + g·8 | AUTH_PASS_CNT / AUTH_FAIL_CNT | RO/W1C | |
| 0xD0 + g·8 | REMOTE_TIME_OFS_LO/HI | RO | From sync seal (SYNC_TYPE 01) |

### 9.8 LHM / FP for Rx n (0x0_3000 + n·0x100)

| Off | Name | Acc | Description |
|---|---|---|---|
| 0x00 | LHM_CTRL | RW | [0] EN, [4:1] EMA k, [5] STATS_RST (SC), [6] TREND_EN, [23:8] TREND_PERIOD (s) |
| 0x04 | LHM_SEL | RW | [3:0] metric ID; write latches readouts |
| 0x08–0x1C | LHM_LAST / MIN / MAX / MEAN / VAR / CNT | RO | Selected metric |
| 0x20 | LHM_LIM_SEL | RW | Metric for limit access |
| 0x24–0x30 | WARN_LO / WARN_HI / ALM_LO / ALM_HI | RW | Limits for selected metric |
| 0x34 | LHM_WEIGHT0..1 | RW | 16 × 4-bit weights packed in 0x34 (metrics 0–7) and 0x38 (metrics 8–15), default 2 |
| 0x3C | LHM_LIM_EN | RW | [15:0] limit check enable per metric |
| 0x40 | HEALTH | RO | [7:0] score, [23:8] WARN bitmap, [31:24] reserved; ALARM bitmap at 0x44 |
| 0x48 | RATE_PPM | RO | Signed |
| 0x4C | MARGIN_TH_HINT | RW | Margin threshold mV (for the DAC driver; informative) |
| 0x60 | FP_CTRL | RW | [1:0] MODE (0 OFF, 1 LEARN, 2 DETECT; reads the effective mode), [5:2] L (4–15), [6] ADAPT, [15:8] D_WORD_TH, [23:16] D_WIN_TH, [27:24] ADAPT_K |
| 0x64 | FP_STATUS | RO | [0] LEARN_DONE, [1] ALARM, [15:8] D_last, [23:16] D_ema (integer), [25:24] learn phase |
| 0x68 | FP_SEL | RW | Feature index |
| 0x6C / 0x70 | FP_MU / FP_INVSIG | RW | Baseline for selected feature (save/restore) |
| 0x74 | FP_ANOM_CNT | RO/W1C | |

### 9.9 Memory regions

- **Scheduler entry (8 B):** +0 DATA (ARINC word), +4 `{EN[31], RSVD[30:24], PERIOD[23:12], OFFSET[11:0]}`.
- **Label table entry (4 B):** §4.5.
- **Mailbox entry (16 B):** +0 DATA (read latches the rest), +4 META, +8 TS_LO, +C TS_HI.
- **Trend entry (64 B):** timestamp, health score, and mean/min/max for up to 8 selected metrics.

---

## 10. Interrupts and event log

- **[IRQ-001]** Per-channel W1C status registers with matching enables. Channel summary bits are in GIRQ_RX/GIRQ_TX. `irq` = OR of enabled pending bits AND GIRQ_EN[0].
- **[IRQ-002]** All interrupt sources shall be level-held until cleared. A clear and a new event in the same cycle shall leave the bit set.
- **[IRQ-003]** Security-relevant events (AUTH_FAIL, SEC_ALARM, SEAL_TIMEOUT, RB_FOREIGN, FP_ALARM, ZEROIZED, KAT_FAIL, rejected non-secure key access) shall also be written to a 64-entry timestamped event log. Log overflow shall be sticky.

---

## 11. Clocking, reset and CDC

- **[CLK-001]** The IP is single-clock (`aclk`, 20–250 MHz). FE2 ADC streams can come from another clock domain. They go through an internal async FIFO (gray-coded pointers), and the sample rate shall be declared in `ADC_RATE` registers.
- **[CLK-002]** `aresetn` is asserted asynchronously and released synchronously inside the IP. Reset value of all outputs: `tx_hi = tx_lo = 0`, `tx_en = 0`, `irq = 0`. These outputs reach their reset values as soon as `aresetn` asserts, also with `aclk` stopped. All outputs are registered; `irq` follows its sources by 1 clk.
- **[CLK-003]** All line inputs are asynchronous and are synchronized inside the IP (`[IF-001]`). The IP shall ship XDC/SDC constraint templates (set_false_path to the first synchronizer flop, ASYNC_REG) and CDC waiver documentation. v1.0 ships `fpga/ip/a429x_constraints.xdc` for Vivado (synchronizer and reset-synchronizer false paths, checked against the routed ZCU106 design), a port-based `fpga/ip/a429x_constraints.sdc` for other tools, and `verif/lint_cdc_waivers.md`.
- **[CLK-004]** Minimum clock for full features: 20 MHz (HS bit = 200 clk). Recommended: ≥ 50 MHz for LHM resolution. Reference: 100 MHz.

---

## 12. Safety, reliability and certification support

- **[SAF-001]** Coding rules: fully synchronous design, no latches, no combinational loops, no internal tri-states, no gated clocks, registered outputs, and one-hot or Hamming-3 FSM encodings with defined recovery to IDLE from illegal states.
- **[SAF-002]** SEU mitigation: SECDED ECC on all BRAMs (`G_ECC_EN`). Configuration registers have parity with background scrub, or TMR (`G_CFG_PROT`). Detected upsets are reported (CFG_SEU), and safety-relevant outputs fall back to a safe state (Tx held NULL) when the policy calls for it.
- **[SAF-003]** Watchdogs: an encoder FSM stuck-timeout and a receiver activity timeout (`RX_ACT_TIMEOUT`) for loss of bus.
- **[SAF-004]** Built-in test: internal loopback for every Tx/Rx pair (`LOOP_INT`; the test is off-bus: while it is set the channel's pins hold NULL and `tx_en = 0`), crypto KAT, and memory BIST on enable (march C− over label table, mailbox and FIFOs, optional).
- **[SAF-005]** The core (non-security) functions shall not depend on SLP or LHM logic. Disabling them at build time removes the logic without changing core behavior (certification partitioning).
- **[SAF-006]** DO-254 kit (DAL A target) deliverables: Hardware Requirements Document (derived from this spec), conceptual and detailed design description, HDL with traceability tags, verification plan, procedures and results, requirements ↔ test traceability matrix, 100 % statement/branch/condition/FSM/toggle coverage report with justified exclusions, elemental analysis, tool assessment notes (simulator, synthesis), configuration index, and accomplishment summary template.
- **[SAF-007]** Security assurance kit (Secure tier): security architecture description, threat analysis aligned with DO-356A/ED-203A methods, crypto implementation description, KAT evidence, and residual-risk statement (§5.13).

---

## 13. Performance and resource targets

### 13.1 Timing targets

| Item | Target |
|---|---|
| Fmax (UltraScale+ -2) | ≥ 250 MHz (all options). v1.0 RTL reaches about 158 MHz (§13.2) |
| Fmax (PolarFire / Artix-7 -1) | ≥ 100 MHz |
| Tx rate error | ≤ 0.01 % |
| Rx word-to-FIFO latency | ≤ 2 µs after bit-32 pulse (WAIT_GAP = 0) |
| Timestamp accuracy | ±2 clk (fixed delay compensated) |
| CMAC (268 bytes) | ≤ 600 clk (measured 390 clk) |
| ENFORCE release after last tag word | ≤ 1 µs + the verifier CMAC time (≤ 600 clk). Measured at 100 MHz: 0.85 µs for a 4-word epoch and 4.25 µs for a 63-word epoch. Starting the verifier CMAC at the seal header would make it ≈ 1 µs for any N (v1.1 option) |

### 13.2 Measured resources (v1.0 RTL, Vivado 2025.2, XCZU7EV-2, out-of-context synthesis, hierarchy kept)

| Block (each instance) | LUT | of which LUTRAM | FF | BRAM36 | DSP |
|---|---|---|---|---|---|
| Rx channel, total (2 seal groups, 1024 mailbox, FIFO 256) | 5,836 | 428 | 4,856 | 7 | 4 |
| ↳ SLP verifier (per seal group, incl. 64 × 160 quarantine) | 1,012 | 184 | 961 | 0 | 0 |
| ↳ decoder (glitch filter, framing, metrics, 4-bit rate classifier) | 1,218 | 0 | 700 | 0 | 2 |
| ↳ channel logic, registers, label table, FIFO, mailbox, bus-load divider | ~2,550 | 60 | ~2,210 | 7 | 2 |
| Shared LHM + fingerprint IDS engine, all 4 Rx channels [LHM-003] | 6,292 | 1,024 | 4,492 | 0 | 3 |
| Tx channel, total (64-entry scheduler, SLP sealer, readback) | 4,126 | 528 | 2,958 | 0 | 4 |
| ↳ RZ encoder (with injection) | 143 | 0 | 150 | 0 | 1 |
| Shared AES-128-CMAC engine (with block prefetch) | 1,819 | 0 | 991 | 0 (10 × RAMB18 for S-boxes) | 0 |
| Core registers, key store (16 × 128-bit in flip-flops for single-cycle zeroize), KAT, KDF, event log | 2,265 | 92 | 4,012 | 0 | 0 |
| Timebase / AXI4-Lite slave / top glue (incl. registered, reset-gated outputs) | 346 / 90 / 442 | 0 | 180 / 155 / 20 | 0 | 0 |
| Virtual Line Emulator (Lab tier only), 4 lines | 1,839 | 0 | 945 | 0 | 20 |
| **Eval build: 4 Tx + 4 Rx, 2 groups, all options, VLE** | **52,942** | 4,940 | **42,104** | 28 + 10 RAMB18 | 55 |
| Same without the VLE (Secure + Sense product build; eval build minus the VLE row) | 51,103 | 4,940 | 41,159 | 28 + 10 RAMB18 | 35 |

The eval build uses 23 % of the XCZU7EV LUTs and 9 % of its flip-flops. It meets 100 MHz with out-of-context WNS +2.38 ns. The worst path is inside the lab-only VLE line model. The product core without the VLE was synthesized against a 4 ns clock. It reaches WNS −2.31 ns, about 158 MHz (synthesis estimate). The 250 MHz UltraScale+ target of §13.1 is therefore **not met in v1.0**. The limiting path is the fingerprint distance accumulator in the LHM engine (feature select and inverse-scale multiply into D), and 660 endpoints fail at 4 ns. Pipelining for 250 MHz is v1.1 work. 100 MHz, the ZCU106 reference clock, has wide margin.

**Area history.** The first build was 81.7k LUTs. Three changes brought it to 49.7k (−39 %):
- LHM statistics and limits moved to LUT-RAM.
- The key store got a single write port.
- One LHM/fingerprint engine is now shared by all Rx channels (−3.4k LUT per Rx).

The security fixes of 2026-10-05 added 1.6k LUTs:
- zeroize and key hygiene in the crypto engine
- fail-secure sealing
- the block-prefetch CMAC

The verification fixes of rev 0.4 added 2.5k LUTs and 7 DSPs:
- FV exhaustion, overflow-safe leap and seal-FIFO tracking in each Tx channel (about 0.4k LUT each)
- the 4-bit-mean rate classifier in every decoder (Rx and readback)
- the span EMA for RATE_PPM and the deferred fingerprint baseline in the LHM engine

Constant multiplies were then rewritten as shift-add sums (×1000, ×10^6, ×16/31, ×31, ÷3) and the small timestamp-compensation product moved to LUTs: −20 DSP (product build 55 → 35) for +0.7k LUT, with no timing change. The remaining DSPs are configuration × configuration products (gap and tolerance thresholds, readback guard time) and true per-word multiplies (statistics variance, fingerprint distance, refresh monitor, encoder skew). Computing the configuration products sequentially at register-write time is a v1.1 option (about −20 more).

**Area roadmap (v1.1).** Target for a 4 + 4 Secure + Sense build is ≤ 35k LUTs. Remaining options:
1. Move the Rx FIFO and quarantine storage to BRAM where depth allows.
2. Share one SLP verifier datapath across groups.
3. Narrow the fingerprint feature arrays.
4. Make the per-channel mailbox, scheduler and readback optional per channel rather than per build.

### 13.3 Crypto engine scheduling bound

Worst case: 16 Tx + 16 Rx channels each request a 17-block CMAC at the same moment. With a round-robin queue and the 600 clk budget per job, the last job finishes ≤ 19,200 clk later (192 µs at 100 MHz). With the measured 390 clk per job it is 12,480 clk (125 µs). That is less than one HS word time (360 µs), and seal words take at least 720 µs to arrive, so `[SLP-033]` and the ENFORCE latency bound hold for every configuration at HS and LS. At faster custom rates they hold while the pending CMAC jobs fit in one word time ([SLP-033]).

---

## 14. Software deliverables

- **[SW-001]** Bare-metal/RTOS C driver (MISRA C:2012 compliant, no dynamic memory). Main API:

```c
int  a429x_init(a429x_t *dev, uintptr_t base, const a429x_cfg_t *cfg);
int  a429x_tx_cfg(a429x_t *dev, unsigned ch, const a429x_tx_cfg_t *c);
int  a429x_tx_send(a429x_t *dev, unsigned ch, uint32_t word);
int  a429x_tx_send_at(a429x_t *dev, unsigned ch, uint32_t word, uint64_t t_ns);
int  a429x_sched_set(a429x_t *dev, unsigned ch, unsigned idx, uint32_t word, uint16_t period, uint16_t offset);
int  a429x_rx_cfg(a429x_t *dev, unsigned ch, const a429x_rx_cfg_t *c);
int  a429x_label_cfg(a429x_t *dev, unsigned ch, uint8_t label, uint8_t sdi, uint32_t entry);
int  a429x_rx_read(a429x_t *dev, unsigned ch, a429x_entry_t *e);       /* data, meta, ts */
int  a429x_mbx_read(a429x_t *dev, unsigned ch, uint8_t label, uint8_t sdi, a429x_entry_t *e);
int  a429x_key_load(a429x_t *dev, unsigned slot, const uint8_t key[16]);
int  a429x_key_derive(a429x_t *dev, unsigned dst, unsigned master, uint16_t bus_id, unsigned grp, unsigned kid);
int  a429x_slp_tx_cfg(a429x_t *dev, unsigned ch, unsigned grp, const a429x_slp_cfg_t *c);
int  a429x_slp_rx_cfg(a429x_t *dev, unsigned ch, unsigned grp, const a429x_slp_cfg_t *c);
int  a429x_fv_get(a429x_t *dev, unsigned dir, unsigned ch, unsigned grp, uint64_t *fv); /* persist */
int  a429x_lhm_get(a429x_t *dev, unsigned ch, unsigned metric, a429x_stat_t *s);
int  a429x_fp_baseline_save(a429x_t *dev, unsigned ch, a429x_fp_base_t *b);
int  a429x_fp_baseline_load(a429x_t *dev, unsigned ch, const a429x_fp_base_t *b);
void a429x_isr(a429x_t *dev);
```

- **[SW-002]** Linux kernel driver (platform driver with device-tree binding `vendor,a429x-1.0`). It exposes a character device per channel with read/write/poll, ioctls for configuration, and sysfs for LHM statistics.
- **[SW-003]** Python reference model `pya429x`: bit-exact encoder and decoder, SLP sealer and verifier (using the `cryptography` package), LHM metric model, and SLP test-vector generator (Appendix A). The same model is the cocotb scoreboard.
- **[SW-004]** Profile tool `a429x-profile`: reads an SLP profile (Appendix B) and the bus traffic description. It checks seal-label conflicts, computes bus load, worst-case latency and tag strength, and generates C init tables for both ends.
- **[SW-005]** Part 3 (Williamsburg) file transfer library on top of the Bulk Transfer Assist.

---

## 15. Verification plan

### 15.1 Environments

| Level | Environment | Purpose |
|---|---|---|
| Unit | cocotb + Verilator / Questa / XSim | Each block against the `pya429x` model |
| Core | cocotb or UVM (SV) with AXI4-Lite VIP and ARINC line BFM | Full register-level scenarios, randomized traffic |
| Formal | SymbiYosys or commercial formal | `tx_hi & tx_lo` never both high; FIFO no-overflow/no-loss; AXI protocol; FSM reachability and recovery; key non-readback (information-flow property) |
| CDC/RDC | Vendor CDC report plus structural review | Synchronizer correctness |
| Gate-level | Post-synthesis netlist sim of a smoke subset | Synthesis/coding mismatches |
| Hardware | ZCU106 (§16) | Silicon validation, interoperability |

### 15.2 Coverage goals

100 % statement, branch, condition, FSM state and transition, and toggle coverage on the RTL, with documented exclusions. Every `[XXX-NNN]` requirement maps to ≥ 1 test. Functional covergroups: all rates × parity modes × gap values × error types; every SLP state transition; every AUTH result code; every TAGW × N corner (1, 63); FV wrap of the 13-bit field; key rollover during traffic.

### 15.3 Test categories (selection)

| ID | Test | Pass criteria |
|---|---|---|
| VT-01 | Tx timing compliance HS/LS/custom at 20, 100, 250 MHz | Rate, pulse width and gap within §4.2 limits |
| VT-02 | Rx tolerance sweep: rate ±1…±15 %, pulse width, glitch widths | Decodes within TOL. Flags outside. |
| VT-03 | Error detection: parity, 31/33-bit, short gap, invalid state | Each detected, counted, flagged |
| VT-04 | Label filter, all 1024 entries, SDI modes | Routing exact |
| VT-05 | Scheduler: 256 entries, overload | Periods exact. SCHED_LATE on overload. |
| VT-06 | Mailbox coherence under concurrent update | No torn reads |
| VT-07 | Refresh monitor stale/fast | Flags within one scanner period |
| VT-10 | SLP golden vectors (Appendix A), all TAGW, N | Bit-exact seal words |
| VT-11 | SLP attacks: inject, modify, replay, delete, reorder, strip, cross-bus splice | Correct FAIL code, no ENFORCE leak |
| VT-12 | SLP FV: 13-bit wrap, window edge, persistence leap, exhaustion | Per §5.7 |
| VT-13 | Zeroize during active traffic | Keys cleared ≤ 64 clk. No further seals. |
| VT-14 | KAT failure injection | Core locked zeroized |
| VT-15 | Legacy coexistence: legacy receiver model filters L_S | Legacy data stream identical with SLP on and off |
| VT-20 | LHM metrics against model with VLE impairments | Within ±2 clk / ±1 LSB |
| VT-21 | FP learn/detect: rogue transmitter with ±20, ±50, ±200 ppm offset | Detection and false-alarm rates measured and reported against [FP-001] |
| VT-22 | Readback: mismatch, lost, foreign | Each flagged |
| VT-30 | SEU injection on config and BRAM | Detected/corrected per §12 |

---

## 16. ZCU106 evaluation platform and lab test plan

### 16.1 Platform facts (from [R9], UG1244)

- Device: Zynq UltraScale+ MPSoC **XCZU7EV**. The PS (quad A53, dual R5F) runs drivers and the test console.
- **PMOD headers J55 (right-angle, female) and J87 (vertical, male):** 3.3 V on the header side. They are level-shifted to FPGA banks 28/66/68 and use **LVCMOS18** on the FPGA side. Maximum interface speed 110 Mb/s, far above the needs of ARINC 429.
- Prototype header J3: 10 more GPIOs.
- Two FMC HPC connectors (HPC0/HPC1). **VADJ is set by the system controller from the FMC card's EEPROM (VITA 57.1). A custom card needs a correctly programmed EEPROM, or VADJ stays off** (see AMD AR 67127 / 67308 for the procedure).
- USB-UART for the console. JTAG for ILA/VIO.

### 16.2 Reference design

```
 +----------------------------- PS (A53 / R5F) -------------------------------+
 |  Vitis bare-metal "a429x-cli" (UART)  |  or PetaLinux + a429x.ko + pya429x |
 +---------+-----------------------------+-----------------+------------------+
   M_AXI_HPM0_FPD          pl_clk0 = 100 MHz         pl_ps_irq0    S_AXI_HP0 (opt DMA)
           |                                                          ^
 +---------v--------------------------- PL ----------------------------+-------+
 | SmartConnect --> A429X eval build: 4 Tx / 4 Rx, AUTH, LHM FE1, FP, RB, VLE  |
 |                       |  Tx0..Tx2 = DUT transmitters                         |
 |                       |  Tx3      = "Attacker" transmitter (Lab)             |
 |                       |  VLE: per-line analog model -> FE0/FE1/FE2 signals   |
 |  AXI DMA (opt) <---- m_axis_rx                                             |
 |  ILA (decoder, SLP verdicts)   VIO (impairment knobs, attack triggers)     |
 |  Pin mux: internal VLE  <->  PMOD J55/J87  <->  FMC PHY card               |
 +----------------------------------------------------------------------------+
```

- **[ZCU-001]** The reference design shall build from a Tcl script (Vivado project mode, block design with IP-XACT-packaged A429X) and a Vitis workspace script. It shall be reproducible from a clean checkout.
- **[ZCU-002]** A run-time pin mux shall select, per channel, between the internal VLE, the PMOD loopback and the external PHY card, with no rebuild needed.

### 16.3 Virtual Line Emulator (VLE): testing analog behavior with no analog hardware

The VLE is a synthesizable model of an ARINC 429 line (Lab tier, `G_VLE_EN = 1`). It lets **all** LHM, FP and attack features be validated on a bare ZCU106:

- Input: `tx_hi`/`tx_lo` from a transmitter. It produces a target differential voltage of +10 / 0 / −10 V in a 12-bit signed sample at 10 mV/LSB, at f_clk.
- Impairments, each set by VIO or AXI:
  - amplitude scale (0.3–1.2)
  - offset and common-mode
  - **slew limiter** (models the driver slope plus cable capacitance; 0.2–20 µs)
  - second-order ringing (overshoot %, frequency)
  - Gaussian-approximate noise (sum of 4 LFSRs, programmable RMS)
  - propagation delay and jitter
  - burst glitches
  - intermittent dropouts (models a fretting connector, with programmable duty and period)
- Comparators with programmable thresholds produce FE0 (`±T_dec`) and FE1 (`±T_margin`) logic signals. The sample stream feeds FE2.
- Attack/contention modes for the attacker transmitter (Tx3, with its NCO offset, for example +150 ppm, and its own slew):
  - CONTEND: voltages summed and clipped
  - GAP_INJECT: the attacker waits for a victim gap and inserts a word
  - MITM: the attacker replaces the victim; replays or modifies recorded words, strips seals
  - REPLAY_BUFFER: stores up to 4096 victim words for later replay

### 16.4 Bring-up phases

| Phase | Extra hardware | What it proves |
|---|---|---|
| **A: Internal** | None | Full functionality through the VLE: protocol, SLP attacks, LHM trends, FP detection |
| **B: Digital loopback** | 8 jumper wires J55 ↔ J87 | Real I/O, synchronizers, level shifters, constraints |
| **C: Physical layer** | A429X-PHY card (§16.5) + cable fixture | True ±10 V signalling, real slew, real comparators (FE0/FE1), readback |
| **D: Interoperability** | A COTS ARINC 429 interface/analyzer and, ideally, a real legacy LRU | Backward compatibility of SLP. Independent timing compliance. |

**Phase B PMOD pin plan (FPGA pins from UG1244 Table 3-33; IOSTANDARD LVCMOS18; confirm header pin numbering against UG1244 Figure 3-31):**

| Net | FPGA pin | Function | | Net | FPGA pin | Function |
|---|---|---|---|---|---|---|
| PMOD0_0 | B23 | TX0_HI | | PMOD1_0 | AN8 | RX0_HI |
| PMOD0_1 | A23 | TX0_LO | | PMOD1_1 | AN9 | RX0_LO |
| PMOD0_2 | F25 | TX1_HI | | PMOD1_2 | AP11 | RX1_HI |
| PMOD0_3 | E20 | TX1_LO | | PMOD1_3 | AN11 | RX1_LO |
| PMOD0_4 | K24 | TX0_SLOPE | | PMOD1_4 | AP9 | RX0_HI_M |
| PMOD0_5 | L23 | TX1_SLOPE | | PMOD1_5 | AP10 | RX0_LO_M |
| PMOD0_6 | L22 | TX_EN (common) | | PMOD1_6 | AP12 | RB0_HI |
| PMOD0_7 | D7 | PPS_OUT (scope trigger) | | PMOD1_7 | AN12 | RB0_LO |

In Phase B, wire TX0_HI→RX0_HI, TX0_LO→RX0_LO, TX1_HI→RX1_HI and TX1_LO→RX1_LO. The margin (RX0_*_M) and readback (RB0_*) inputs may be jumpered from the same TX0 nets. FE1 then reports ideal values, which exercises the logic but not the analog measurement. The PMOD translators are auto-direction types (check the part in the UG1244 schematic), so avoid strong pull-ups or pull-downs on these nets.

### 16.5 A429X-PHY evaluation card (to be designed)

- **Form factor:** FMC (LPC subset, VITA 57.1 EEPROM programmed for VADJ = 1.8 V) for 4 Tx + 4 Rx + 4 readback. A reduced 1 Tx / 1 Rx version can go on PMOD (3.3 V logic, external supply for the driver).
- **Tx:** a single-supply ARINC 429 line driver with slope control (for example a Holt HI-859x-class part; confirm availability and data sheet). Logic-level DATA HI/LO, SLOPE and enable inputs.
- **Rx (FE0):** quad ARINC 429 line receiver (for example a Holt HI-845x-class 3.3 V part).
- **FE1 margin:** per input, a resistive divider plus 2 fast comparators. Thresholds come from a quad I²C DAC (±6–9.5 V equivalent range), so margin thresholds are software-set.
- **FE2 option (v1.1):** dual-channel 12-bit ≥ 10 MSPS ADC per receiver on a daughter variant.
- **Readback:** a second receiver tied to each Tx output.
- **Protection:** TVS on line pins and lightning-induced transient footprint (for lab use only; DO-160 qualification is not a goal of the eval card).
- **Cable and fault fixture ("Line Torture Box"):**
  - Selectable shielded twisted pair lengths: 1 m, 30 m, 100 m spool.
  - Series resistance steps 0–100 Ω (corrosion / pin fretting).
  - Shunt leakage 10 kΩ–1 MΩ (moisture).
  - Added shunt capacitance 0–10 nF (crushed cable).
  - Relay-driven intermittent open (chatter, 1–100 ms).
  - Receiver load bank emulating 1–20 ARINC 429 receivers.
  - A second "rogue" driver that can be bridged onto the pair.

### 16.6 Lab test cases on ZCU106

| ID | Phase | Test | Pass criteria |
|---|---|---|---|
| ZT-01 | A | Internal loopback, all channels, HS/LS/AUTO, 10^8 words | 0 errors. Counters match. |
| ZT-02 | A | SLP default profile (N=16, TAGW=2), ENFORCE, 24 h soak | 0 false failures |
| ZT-03 | A | Attack suite via VLE (inject, modify, replay, strip, MITM) | 100 % detected. 0 forged words released in ENFORCE. |
| ZT-04 | A | Gap injection by attacker at +300 ppm (FE0, HS, 100 MHz; per-word resolution 32 ppm) | FP_ANOMALY on ≥ 99 % of injected words. False positives < 10^-4 per legitimate word. |
| ZT-05 | A | Degradation sweep: slew 1.5→6 µs, amplitude 1.0→0.7 | LHM WARN before first decode error, in 100 % of sweeps |
| ZT-06 | A | Zeroize pin / KAT failure | Per VT-13/14 |
| ZT-10 | B | External PMOD loopback at HS/LS, rate tolerance sweep | Same results as ZT-01. Timestamps within ±2 clk. |
| ZT-20 | C | Electrical compliance with oscilloscope: rate, pulse, rise/fall, gap | Within §4.2 |
| ZT-21 | C | Torture box: series R and shunt C steps, intermittent open | LHM trend detects each step. Health score falls monotonically. |
| ZT-22 | C | Rogue driver bridged onto pair during gaps | RB_FOREIGN at the DUT Tx. FP/SLP alarms at the DUT Rx. |
| ZT-30 | D | COTS ARINC 429 analyzer as legacy receiver of a sealed bus | All non-seal labels received unchanged. Seal label shows as ordinary words. No errors. |
| ZT-31 | D | COTS transmitter → A429X Rx (OFF, MONITOR) | Correct decode. MONITOR reports UNSYNCED/timeout only for configured PLS. |
| ZT-32 | D | Timing cross-check A429X Tx vs COTS analyzer measurements | Agreement within the analyzer's resolution |

### 16.7 Demonstrations (sales collateral)

1. **"Spoofed altitude":** a legacy receiver displays a fake altitude injected by the attacker. On the same bus, the A429X receiver in ENFORCE rejects it, and the GUI shows AUTH_FAIL and FP_ANOMALY with timestamps.
2. **"The wire that was going to fail":** the torture box steps series R and shunt C upward. The health-score trend falls and raises WARN while data integrity is still 100 %, minutes before the first parity error.
3. **"Zero-change retrofit":** a COTS analyzer standing in for an untouched legacy LRU keeps receiving every label normally while the bus is sealed.

---

## 17. Product deliverables and licensing

| Deliverable | Core | Secure | Sense | Complete |
|---|---|---|---|---|
| RTL (VHDL-2008 *or* SystemVerilog; encrypted IEEE 1735 or source license) | ✓ | ✓ | ✓ | ✓ |
| IP-XACT package for Vivado IP Integrator; templates for Libero, Quartus, Radiant, Efinity | ✓ | ✓ | ✓ | ✓ |
| Product guide, register map (also as IP-XACT/SystemRDL → C headers) | ✓ | ✓ | ✓ | ✓ |
| Bare-metal and Linux drivers, `pya429x` model | ✓ | ✓ | ✓ | ✓ |
| Verification environment and regression results | ✓ | ✓ | ✓ | ✓ |
| DO-254 DAL A certification kit | option | option | option | option |
| SLP profile tool, security assurance kit | | ✓ | | ✓ |
| LHM default limits, analysis tool (trend viewer) | | | ✓ | ✓ |
| ZCU106 reference design (bitstream + sources) | ✓ | ✓ | ✓ | ✓ |
| A429X-PHY card design files (schematic/layout) | option | option | option | option |

- Licensing model suggestions: per-project or site license, plus royalty-free use; or a lower upfront fee with per-unit royalty. Certification kit and support priced separately. Evaluation: a time-limited netlist for the ZCU106.
- **Export control:** the Secure tier contains cryptography. Obtain an export classification (for example EAR Category 5 Part 2 review) before shipping outside the home jurisdiction.

---

## 18. Roadmap

| Version | Content |
|---|---|
| v1.0 | Core + SLP (CMAC) + LHM FE0/FE1 + FP + Readback; ZCU106 phases A–D |
| v1.1 | FE2 analog front-end, Ascon option, trend viewer GUI, Linux DMA path |
| v1.2 | Hardware ARINC 429 Part 3 (Williamsburg) engine |
| v2.0 | **In-situ wire fault location**: spread-spectrum TDR injected below the receiver NULL threshold during gaps, reported as distance-to-fault (needs an analog injection path on the PHY) |
| v2.x | Pursue an industry-standard form of SLP (for example propose it as a supplement or project paper through SAE ITC / AEEC) to turn the protocol into an ecosystem |

---

## 19. Risks, open issues and legal notes

| # | Item | Mitigation / action |
|---|---|---|
| R1 | **Prior art / freedom to operate.** Truncated-MAC + truncated-freshness is known from automotive SecOC. Academic work exists on ARINC 429 authentication and on hardware fingerprinting of ARINC 429 transmitters (for example Gilboa-Markevich & Wool, ESORICS 2020). | Run a professional patent search before making novelty claims. Any patentable novelty is likely in the specific combination: seal-label backward compatibility, epoch quarantine with ENFORCE/EARLY modes, FP-gated SLP policy, and margin-comparator LHM. |
| R2 | Seal-label allocation varies by aircraft and ICD | Profile tool checks conflicts. The IP is label-agnostic. Standardization effort (§18). |
| R3 | Bus load headroom on heavily loaded legacy buses | Selective PLS, large N, EARLY mode. The profile tool flags over-budget buses. |
| R4 | Legacy receivers behaving unexpectedly with unknown labels | Interop test ZT-30 on representative LRUs. Report behavior per LRU. |
| R5 | System key management is outside the IP | Ship a reference key-management concept (factory master key + per-bus KDF + KEK wrap) as an application note. |
| R6 | Replay after receiver reboot (§5.13) | Documented residual risk, time-checked sync option |
| R7 | LHM thresholds need field data | Conservative defaults. Trend buffer to collect fleet data. Partnering with an MRO for a pilot. |
| R8 | FP false alarms under temperature extremes | Adaptive baseline (`FP_ADAPT`), qualification in a thermal chamber (Phase C) |
| R9 | Side-channel attacks on keys (DPA) in hostile-access scenarios | Optional masked AES implementation in a later version. Document the threat assumption. |
| R10 | ARINC specification text is copyrighted | Do not reproduce it. Reference clause numbers only. |

**Open questions for the next revision:**
1. HDL language for the product (VHDL-2008 is more common in DO-254 shops; SystemVerilog is better for UVM customers). Is a dual deliverable worth the cost?
2. Is 16 + 16 channels enough, or is a "32-channel concentrator" SKU needed for IMA/data-concentrator customers?
3. Should FE1 thresholds be standardized (for example decision ±4.5 V, margin ±8.0 V) to simplify the PHY card and default limits?
4. Default SLP profile per data class (display data vs. control data) to be defined with a pilot customer.

---

## 20. v1.0 RTL implementation status (2026-10-05)

### 20.1 Implemented and verified

| Area | Status |
|---|---|
| Tx: NCO bit rate, RZ encoder, HI/LO interlock, parity modes, gap, FIFO, timed transmit, error injection (all 7 codes [TX-015], ignored outside TEST_MODE) | Implemented, verified |
| Tx: 64/256-entry periodic scheduler, FREEZE, late detection | Implemented, verified |
| Tx: SLP sealer (4 groups max, N 1–63, T_EPOCH, TAGW 1–3, KID 0–3, sync seals FV / FV+TIME, FV leap, persist/exhaust events) | Implemented, verified |
| Tx: fail-secure sealing [SLP-034]: protected words blocked when they cannot be sealed, open epoch abandoned on key loss | Implemented, verified |
| Tx: readback compare, lost, foreign-transmitter detection | Implemented, verified |
| Rx: glitch filter, decoder, frame/gap/rate/parity/invalid detection, auto-rate (HS/LS/unknown), timestamps, bus load | Implemented, verified |
| Rx: 1024-entry label table, mailbox (256/1024), refresh monitor (stale/too-fast), FIFO with META | Implemented, verified |
| Rx: SLP verifier (quarantine, check order [SLP-041], FV reconstruction/window/13-bit carry, MONITOR/EARLY/ENFORCE, verdict records, K_ALARM, alarm policy, time check, FP policy, mailbox protection) | Implemented, verified |
| LHM FE0/FE1 metrics, EMA statistics, limits, health score, RATE_PPM; one engine shared by all Rx channels [LHM-003] | Implemented, verified |
| Fingerprint IDS learn (mean + MAD) / detect, per-word flag, window alarm, drift adaptation (α = 2^−ADAPT_K) | Implemented, verified |
| AES-128-CMAC shared engine: round-robin, next-block prefetch (390 clk for the longest MAC), key hygiene (no key-derived value held while idle) | Implemented, verified |
| Key store (16 slots, single write port), secure-bus key access [KEY-003], power-up KAT (3 RFC 4493 vectors) with failure lockout [KEY-006] | Implemented, verified |
| Zeroize [KEY-005]: register and glitch-filtered pin, wipes slots, staging, subkeys, chaining value, tag and AES state; cancels jobs in flight (incl. key derivation) | Implemented, verified |
| Key derivation command (SP 800-108r1 CMAC counter-mode KDF), checked against the `cryptography` library | Implemented, verified |
| FV monotonic across disable/enable (max(written, last) + leap) | Implemented, verified |
| AXI4-Lite slave, register map §9, interrupts (clear/set race [IRQ-002]), 64-entry security event log | Implemented, verified |
| Virtual Line Emulator (Lab tier) | Implemented, verified |

### 20.2 Deferred (roadmap, §18) and known deviations

- **Not implemented:**
  - Ascon option
  - AES key wrap (UNWRAP returns CMD_ERR)
  - FE2 ADC front end
  - LHM trend ring buffer
  - `K_RATE` sliding-window alarm [SLP-044]
  - Bulk Transfer Assist / Part 3 hardware
  - AXI4-Stream data ports
  - ECC and TMR options
  - Readback LHM metrics: the RB region exposes counters only.
  - `GCTRL.EN` and `FREEZE_ALL` (reserved)
  - LHM default limit presets [LHM-032]
  - Linux kernel driver [SW-002]. v1.0 ships a UIO user-space library and CLI (`sw/linux`) instead.
- **Partial:**
  - `GCTRL.LOCK` locks GCTRL only. The full configuration lock is v1.1.
  - SLVERR is generated by the core register block only [IF-002].
  - Memory BIST on enable [SAF-004] (optional) is not implemented; internal loopback and the crypto KAT are.
- **[SAF-001]/[SAF-003] deviations, scheduled for v1.1:**
  - FSMs use binary encoding, each with a default recovery branch, not one-hot or Hamming-3. Illegal codes recover to IDLE, but a single-bit upset can land in another legal state. Fault injection on the Tx path showed one stale word per upset in the encoder, arbiter and scanner; an upset into the sealer's CMAC wait state locks the channel until `TX_CTRL.EN = 0` (the [SAF-003] watchdog covers this in v1.1), and an upset into its emit state sends spurious seal words (`sim/run_tx_clk_sweep.ps1 -SafEmit`).
  - The encoder and receiver watchdogs are not implemented.
- **Clock floor:** the refresh scanner needs f_clk ≥ 41 MHz to meet 100 µs with 1024 mailbox entries [RX-032].
- **Fmax:** about 158 MHz on UltraScale+ -2 (synthesis estimate, product core), against the 250 MHz target in §13.1. The limiting paths are in the shared LHM engine (fingerprint distance accumulator, statistics datapath). Pipelining is planned for v1.1, and 100 MHz has wide margin.

### 20.3 Verification results

All results below come from `sim/regress.ps1` (13 benches and 3 clock-variant runs, plus the Python model self-tests and the profile-tool unit tests) with the embedded assertions enabled. Every run passed with 0 violations. Lines marked `DEVIATION` report the documented [SAF-001] FSM-encoding deviation (§20.2) and are not counted as failures.

| Bench | Scope | Result |
|---|---|---|
| `tb_cmac` | FIPS-197 AES; RFC 4493 CMAC on the K1 and K2 paths; two concurrent requesters; no key material after a job; zeroize mid-job (abort broadcast, no tag, wipe in 1 clk, recovery) | ALL PASS |
| `tb_a429x_min` | Core tier (1 Tx / 1 Rx, no SLP, LHM, scheduler or mailbox): data integrity, absent regions, DECERR | ALL PASS (20 checks) |
| `tb_a429x` | Full eval build over AXI4-Lite only (T0–T18), see below | ALL PASS (196 checks) |
| `tb_a429x_sec` | Security, small build with `G_SECURE_BUS = 1` (S0–S14), see below | ALL PASS (104 checks) |
| `tb_a429x_phy` | Physical layer (P1–P6), see below | ALL PASS (46 checks) |
| `tb_a429x_soak` | 4 Tx + 4 Rx busy at once, 2 seal groups (TAGW 2 and 3), randomized traffic, error injection, scoreboards per channel and group | ALL PASS (2409 checks) |
| `tb_a429x_equiv` | Full build and core build side by side on the same traffic: SLP off leaves ARINC 429 behaviour unchanged [SAF-005] | ALL PASS (159 checks) |
| `tb_a429x_drv` | The C driver `sw/a429x` compiled into the simulation (DPI-C) and run against the RTL: init, Tx/Rx, keys, KDF, FV, LHM/FP, zeroize | ALL PASS (29 checks) |
| `tb_a429x_slp_tx` | Sealer bit-exact against `pya429x --golden2` (TAGW 1–3, N 1/63, sync types, both groups, KID 0/1/3, 13-bit wrap, FV 2^48 − 2), FV exhaustion and fail-secure blocking, leap, persistence, sync schedule, epoch closing and contiguity on 4 channels, key-command interlocks, legacy-receiver transparency at HS and LS | ALL PASS (388 checks) |
| `tb_a429x_slp_rx` | Verifier: sync seals and TIME check (also at absolute PTP time), quarantine per mode, filter-independent quarantine, alarm policy, FP policy, line corruption, replay, zeroize/UNSYNCED, event log types and overflow | ALL PASS (301 checks) |
| `tb_a429x_tx` | Scheduler (FREEZE, phases, 256 entries, priority, late), gap, FIFO overflow, timed release, parity modes, slope, counters, IRQ gating, off-bus loopback, reset with clock stopped, AXI responses and channel ordering (back-to-back, W/AW order, AR with AW+W, held B/R, partial WSTRB), FSM upset injection | ALL PASS (222 checks; 4 SAF-001 deviations) |
| `tb_a429x_rx` | Latency (also 4 channels with LHM), timestamps, framing, glitch/invalid, error storage, overflow, SEQ wrap, bus load, label table, mailbox coherence, auto-rate windows, timebase and PPS | ALL PASS (158 checks) |
| `tb_a429x_rx` at 20 MHz | `sim/run_rx_20mhz.ps1`: the clock-sensitive subset at the 20 MHz floor | ALL PASS (54 checks) |
| `tb_a429x_soak` at 20 MHz | `sim/run_20mhz.ps1`: the soak traffic (SLP sealing and ENFORCE on 2 groups, error injection, scheduler, loopback, LHM on 4 channels) at the 20 MHz floor | ALL PASS (2412 checks) |
| `tb_a429x_tx` clock sweep | `sim/run_tx_clk_sweep.ps1`: bit rate and edge placement at 20, 33.333, 100, 150 and 250 MHz, HS and LS (worst 0.63 ppm, ≤ 1 clk) | 5 of 5 ALL PASS |
| `tb_a429x_lhm` | Every LHM metric against a pin-level reference, statistics, limits, health score, RATE_PPM, fingerprint learn/restore/detect/adapt/alarm against `model/lhm_model.py`, measured detection table ([FP-001]) | ALL PASS (240 checks) |
| `tb_gls` | Gate-level netlist from out-of-context synthesis: KAT, loopback, SLP golden | ALL PASS (24 checks) |
| Parameter sweep | Elaboration of core-only 1×1, 1 group with 256 mailbox and 256 scheduler entries, 16 Tx × 16 Rx × 4 groups | All elaborate |
| Embedded assertions (`A429X_ASSERT`) | Every clock in all benches: [TX-002] HI/LO interlock, [TX-008] NULL while disabled, [TX-005] minimum gap, AXI response hold and exclusivity, [KEY-005] idle crypto engine holds no key material | 0 violations |
| `tb_sva_probe` | Assertion validity: zero false reports on an active and an idle channel, and correct detection when HI+LO are forced on the idle channel only | Pass |

`tb_a429x` covers:
- HS timing compliance (pulse/bit/gap error 0 ns)
- data integrity, filtering, error injection, mailbox, scheduler
- SLP seals **bit-exact against the independent Python model**
- injection, replay, forgery and stripping attacks, with recovery
- fingerprint IDS (attacker +1 %, 0 false positives) and drift adaptation
- LHM slew and amplitude degradation through the VLE
- zeroize and the event log
- hardware key derivation sealing verified with the model-derived key
- registers, IRQ and timebase

`tb_a429x_sec` covers:
- KAT failure lockout
- non-secure key access (rejected, logged, SLVERR)
- TAGW 1 / KID 1 in ENFORCE and EARLY, with every verdict field checked
- mailbox untouched by failed epochs
- six kinds of malformed or forged seal, in check order
- FV window edge W / W+1, K_ALARM → UNSYNCED, resync, floor replay, 13-bit carry
- zeroize by register, by pin (glitches ignored) and during key derivation: all crypto registers zero, no protected word or seal transmitted afterwards
- key invalidated mid-epoch
- BYPASS_SEAL honored only in TEST_MODE
- W1C/event race at −2…+2 clk
- 63-word epoch: 390 clk sealer and verifier
- readback one-bit mismatch and lost readback

`tb_a429x_phy` covers:
- all injection codes ignored outside TEST_MODE
- skew of +63, −64 and +16 /512 with exact periods and RATE_ERR detection
- glitch widths of 80, 160 and 320 ns, and the receiver glitch filter
- HI/LO swap
- auto-rate HS → LS → 20 kbps (UNKNOWN), with RATE_CHANGE and decoding
- refresh monitor: TOO_FAST, and STALE within one scan period

**Code coverage**, merged over the 13 regression benches (`sim/coverage.ps1`): line 95.5 %, branch 80.6 %, condition 77.1 %, toggle 29.2 % (35 module variants). The six-bench merge of rev 0.3 scored 95.7 / 80.9 / 77.2 / 35.1 %: the new benches add builds with other parameter sets (for example 1 Tx / 1 Rx, 20 MHz, no mailbox), and xcrg scores each parameter variant of a module separately instead of merging it, so the new, partly exercised variants lower the averages. Per-module figures are in `sim/coverage/modules_summary.md`.
- Toggle coverage is low because wide data paths are never fully toggled by directed tests: 48-bit FV, 64-bit timestamps, 128-bit keys and RAM data.
- `xcrg` reports module variants with other parameter sets separately, for example the small builds of `tb_a429x_min`, `_sec` and `_phy`.
- Closing the §15.2 goal (100 % with justified exclusions) is open work.

The same properties are provided as concurrent assertions under `A429X_FORMAL` for formal tools. Simulation uses immediate (procedural) assertions because xsim 2025.2 mis-evaluated the concurrent form inside the second and later instances of a generate loop. The probe bench reproduces this: an idle channel whose outputs never left 0 reported a failure every cycle.

**Out-of-context synthesis** (Vivado 2025.2, XCZU7EV-2, eval configuration 4 Tx + 4 Rx, 2 groups, VLE): the results are:
- 52,942 LUT (23 %), 42,104 FF (9 %), 28 RAMB36 + 10 RAMB18, 55 DSP
- WNS +2.38 ns at 100 MHz; the worst path is in the VLE
- 54 `ASYNC_REG` synchronizer flip-flops cover every asynchronous input; the methodology report gives advisory warnings only
- the product core without the VLE reaches about 158 MHz (§13.2)

**ZCU106 hardware build** (eval build inside the Zynq UltraScale+ block design, final RTL):
- Timing met at 100 MHz: WNS +1.404 ns, WHS +0.010 ns, 0 failing endpoints of 165,146. The worst path is in the lab-only VLE. The AXI write-data register carries `max_fanout = 64`: without it one placement left a 9.3 ns pure-route path to a far Tx channel (WNS +0.44 ns).
- Utilization: 53,012 LUT (23.0 %), 43,199 FF (9.4 %), 33 BRAM, 55 DSP.
- Bitstream and XSA exported. The Vitis A53 platform is updated to this XSA, and the self-test app (H0–H12) builds with 0 warnings in A429X code.
- On-board run (§16, Phase A), 2026-10-06: the self-test H0–H12 passes 32 of 32 checks (`A429X_HW_RESULT: ALL PASS`, log in `sw/zcu106_selftest/results/`): identity and KAT, 100 kbps loopback, the VLE line path, SLP seals bit-exact with the Python model, injection / replay / stripping rejected with ALARM and recovery, the +200 ppm fingerprint attacker flagged with no false positives, slew and amplitude degradation detected, zeroize, hardware key derivation and fail-secure blocking.
- Bring-up findings (no RTL change): (1) H8 overfilled the 64-word Tx FIFO; `a429x_tx_send()` now returns -1 on a full FIFO (as the Linux library already did) and `a429x_tx_send_wait()` was added. (2) The board's PS DDR SODIMM uses x16 devices while the ZCU106 board preset configures x8 devices, so address bit 14 (BG1) aliases; the self-test now runs from OCM, and a DDR-based application needs the PS DDR configuration set to the fitted module. (3) `run_on_board.tcl` selects JTAG as the alternate boot mode, so the board's QSPI image cannot start during the download.

**Portability:** Efinity 2026.1 synthesizes the same RTL for Titanium Ti375 (C4) with 0 errors: 108.6k LUT4, 63.5k FF, 93 DSP (85 DSP48 + 8 DSP24), 205 RAM10 (2026-10-06, map stage). Place-and-route on Efinix has not been run.

**IP-XACT:** the Vivado IP package (`fpga/ip`) passes validation in a block design. Parameters propagate, the address range is 512 KB, and IP synthesis completes with 0 errors.

**Design defects found and fixed during verification:**
1. The crypto word-read pipeline sampled RAM one cycle early. Tx and Rx were affected identically, so only the independent golden model exposed it.
2. The decoder interval timers were off by one clock.
3. In AUTO rate mode, the gap threshold lagged the per-word period estimate by one clock, so every word ended after bit 1. Found by `tb_a429x_phy`.
4. Zeroize did not clear the CMAC subkeys, chaining value, last tag or the derived-key register. Found in the traceability review; fixed and checked by assertion and `tb_a429x_sec`.
5. A protected word was sent unsealed when the transmitter could not seal it. Fixed per [SLP-034].

## Appendix A: SLP worked example and test-vector format

### A.1 Example bus

An air-data-style transmitter at HS sends 12 labels totaling **300 words/s** (10.8 % load).

| Option | Config | Seal words/s | Total load | Max added latency (ENFORCE) |
|---|---|---|---|---|
| Protect all, count-driven | N = 16, TAGW = 2 | 300/16 × 3 ≈ 56 | 356 w/s = 12.8 % | ≈ 53 ms (16 words at 300 w/s) + 1.1 ms |
| Protect all, latency-bounded | N = 63, T_EPOCH = 20 ms, TAGW = 2 | 50 seals/s × 3 = 150 | 450 w/s = 16.2 % | ≤ 20 ms + 1.1 ms |
| Selective: 4 critical labels at 100 w/s | N = 8, TAGW = 3 | 100/8 × 4 = 50 | 350 w/s = 12.6 % | ≤ 80 ms or T_EPOCH |
| Any of the above, EARLY mode | — | same | same | 0 ms (verdict follows) |

### A.2 Test-vector file format (`slp_vectors.jsonl`, one vector per line)

```json
{"id":"SLP-TV-0007","key":"2b7e151628aed2a6abf7158809cf4f3c","bus_id":"0x1A2B","group":0,
 "seal_label":"0o3X5","kid":1,"tagw":2,"fv":"0x00000001F3A0","words":["0x...","0x..."],
 "expected_seal":["0x...","0x...","0x..."],"expected_rx":"PASS"}
```

`pya429x` generates the vectors. Each release regenerates them and cross-checks against an independent CMAC implementation (OpenSSL) and the RFC 4493 vectors.

---

## Appendix B: SLP profile descriptor (input to `a429x-profile`)

```yaml
profile: ADC1_BUS_A
bus_id: 0x1A2B
rate: HS
groups:
  - group: 0
    seal_label: 0o3X5          # must be unused in the ICD (tool checks)
    tagw: 2
    n: 16
    t_epoch_ms: 20
    sync_period_s: 1
    sync_type: FV_TIME
    kid_active: 0
    pls:                        # (label, sdi) pairs; sdi '*' = all
      - [0o203, '*']
      - [0o206, '*']
      - [0o210, '*']
receivers:
  - name: FMS1   ; mode: ENFORCE ; win_log2: 10 ; k_alarm: 3
  - name: DFDR   ; mode: MONITOR
legacy_receivers: [AFCS1, EGPWS]   # tool asserts seal label not in their ICD
traffic:                           # for load and latency computation
  - [0o203, 16]                    # label, Hz
  - [0o206, 16]
  - [0o210, 8]
```

Outputs: bus load before and after, worst-case latency per receiver mode, tag strength, conflicts, and generated `a429x_profile_ADC1_BUS_A.h` for both ends.

---

## Appendix C: Glossary

| Term | Meaning |
|---|---|
| BNR / BCD | Binary / binary-coded-decimal ARINC data encodings |
| CMAC | Cipher-based MAC (NIST SP 800-38B) |
| EMA | Exponential moving average |
| FE0/FE1/FE2 | Front-end tiers (§6.2) |
| FP | Physical Fingerprint IDS |
| FV | Freshness Value |
| ICD | Interface Control Document |
| LHM | Line Health Monitor |
| LRU | Line Replaceable Unit |
| NFF | No Fault Found |
| PLS | Protected Label Set |
| RZ | Return-to-zero |
| SDI / SSM | Source/Destination Identifier / Sign-Status Matrix |
| SHW / STW / SYW | Seal Header / Tag / Sync word |
| SLP | Sealed Label Protocol |
| VLE | Virtual Line Emulator |
