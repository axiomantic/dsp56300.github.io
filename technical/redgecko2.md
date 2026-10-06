---
title: "Red Gecko 2 (Nord Modular G2) Technical Information"
layout: default
permalink: /technical/redgecko2
---

```
██████╗ ███████╗██████╗      ██████╗ ███████╗ ██████╗██╗  ██╗ ██████╗     ██████╗ 
██╔══██╗██╔════╝██╔══██╗    ██╔════╝ ██╔════╝██╔════╝██║ ██╔╝██╔═══██╗    ╚════██╗
██████╔╝█████╗  ██║  ██║    ██║  ███╗█████╗  ██║     █████╔╝ ██║   ██║     █████╔╝
██╔══██╗██╔══╝  ██║  ██║    ██║   ██║██╔══╝  ██║     ██╔═██╗ ██║   ██║    ██╔═══╝ 
██║  ██║███████╗██████╔╝    ╚██████╔╝███████╗╚██████╗██║  ██╗╚██████╔╝    ███████╗
╚═╝  ╚═╝╚══════╝╚═════╝      ╚═════╝ ╚══════╝ ╚═════╝╚═╝  ╚═╝ ╚═════╝     ╚══════╝
```

# Red Gecko 2 (Nord Modular G2) Technical Information

All technical information we discovered on the architecture of the Clavia Nord Modular G2 synthesizer family is documented here. Information was gathered through schematic recovery, firmware disassembly, hardware test points, and building a cycle-accurate emulator in Gearmulator.

## Hardware Overview

The Clavia Nord Modular G2 (2004) is a hardware-accelerated modular synthesizer. Unlike fixed-architecture virtual analog synths (such as the Waldorf microQ, Access Virus, or Nord Lead series), the G2 executes user-constructed modular synthesis graphs compiled into DSP machine code routines and transferred over USB in real time. The system combines a **Motorola MCF5407 ColdFire V4 32-bit microcontroller** with an array of up to **eight Motorola DSP56367 24-bit digital signal processors**. The ColdFire MCU acts as the central conductor: managing USB, physical MIDI I/O, front panel controls, preset storage, and distribution of compiled DSP routines across an 8-channel parallel host bus.

### Hardware Variants

- **G2 (Keyboard)** — 3-octave keyboard with aftertouch, 5 graphical LCDs, 8 rotary encoders with 15-LED rings, pitch stick, mod wheel, 4× DSP56367 baseboard (expandable to 8).
- **G2X (Keyboard)** — 5-octave keyboard with aftertouch, dual mod wheels, pitch stick, 5 graphical LCDs, factory-fitted expansion board (8× DSP56367 total).
- **G2 Engine (1U Rack)** — 1U rackmount without front panel controls, LCDs, or keyboard. Controllable via USB and MIDI, 4× DSP56367 baseboard (expandable to 8).

### Key Specifications

| Subsystem | Specification |
|---|---|
| **Host Microcontroller** | Motorola MCF5407CAI162 (ColdFire V4, 32-bit) |
| **MCU Core Clock** | 162.0 MHz nominal (catalog rating; see clock distribution) |
| **DSP Array** | Up to 8× Motorola DSP56367 (24-bit fixed-point) |
| **DSP Core Clock** | 147,456,000 Hz (147.456 MHz crystal) |
| **Audio Sample Rate** | 96,000 Hz (96 kHz native frame rate) |
| **DSP Cycles per Frame** | Exactly 1,536 DSP clock cycles per audio frame |
| **Control Frame Rate** | 24,000 Hz (1 control frame every 4 audio frames; 6,144 cycles) |
| **Audio I/O** | 4 balanced inputs, 4 balanced outputs (24-bit / 96 kHz) |
| **Host ↔ DSP Bus** | CS1 HDI08 parallel bus (active-low one-cold select) |
| **System SDRAM** | 8 MiB SDRAM window mapped at `0x30000000` |
| **Firmware & Patch Flash** | 8 MiB NOR Flash window mapped at `0x12000000` (CS2) |
| **Boot Flash** | 512 KiB NOR Flash mapped at CS0 / CSBOOT (`U21`) |
| **USB Controller** | Philips ISP1181ADGG (`U24`) Full-Speed USB controller |
| **Panel Displays** | 5× multi-cell graphical LCDs (quiescent CS4 buffer in emulation) |
| **Panel Controls** | 5 scanned analogue inputs via Maxim MAX1039 I²C ADC; encoders |

### Architectural Comparison with Sibling Synthesizers

Within the Gearmulator ecosystem, the G2 represents the most computationally demanding design:

| Feature | Nord Modular G2 (Red Gecko 2) | Waldorf microQ (Vavra) | Nord Lead 2X (NodalRed2x) | Waldorf Microwave II/XT (Xenia) |
|---|---|---|---|---|
| **Host CPU** | Motorola MCF5407 (ColdFire V4 @ 162.0 MHz) | Motorola MC68331 (68300 @ 16–25 MHz) | Motorola MC68331 (68300 @ 16.0 MHz) | Motorola MC68331 (68300 @ 16.0 MHz) |
| **DSP Array** | 4–8× Motorola DSP56367 | 1–3× Motorola DSP56362 | 2× Motorola DSP56362 | 1–2× Motorola DSP56303 |
| **DSP Clock** | 147.456 MHz | 101.606 MHz | 100.0 MHz | 100.0 MHz |
| **Sample Rate** | 96,000 Hz | 44,100 Hz | 98,200 Hz | 44,100 Hz |
| **Synthesis Model** | Dynamic graph (PRAM injection) | Static ROM + expansion | Dual-DSP split pipeline | Static ROM wavetable |
| **Host Bus** | CS1 HDI08 (One-cold select) | Discrete HDI08 ports | Shared / discrete HDI08 | Discrete HDI08 ports |
| **Audio Bus** | ESAI Serial TDM Network | ESAI 20-slot TDM Ring | ESAI Point-to-Point (2× rate) | ESSI Serial Ring |
| **Emulation** | `mcf5407` (Clean-room C11) + `rg2Lib` | Musashi MC68k + `synthLib` | Musashi MC68k + `synthLib` | Musashi MC68k + `synthLib` |

## Motorola MCF5407 Microcontroller

The Motorola MCF5407 is a 32-bit microprocessor based on the ColdFire Version 4 (V4) core architecture, featuring a dual-issue execution pipeline with branch prediction and hardware integer multiply/divide units. The MCU runs the RTOS that parses `.pch2` patches, manages DSP memory, and schedules control frames across the DSP cluster.

### Architecture & Clock Distribution

- **Core Architecture**: Motorola ColdFire Version 4 (V4) core (*MCF5407UM/D* section 1.1). The *ColdFire Family Programmer's Reference Manual (CFPRM)* treats MMU and FPU as optional per-device modules reported in D0 bits 11 and 12 at reset; "V4e" designates V4 with MMU and FPU (*MCF5485EC* Rev. 4).
- **Oscillator Can**: Physical crystal oscillator can on the mainboard is rated **53.620 MHz**.
- **Schematic Net Label**: The schematic identifies the clock net entering the MCU pin as `CPU Ck 54MHz`.
- **Core Clock**: 162.0 MHz is modeled in the emulator, matching nominal speed grade. A 3× PLL multiplier on 53.620 MHz yields 160.860 MHz (0.7% discrepancy). Without physical oscilloscope verification on CLKIN, operating frequency sits within this 0.7% window.
- **Bus Clock (BCLKO)**: Bus clock divider is unmeasured on hardware and underived in the emulator (`G2_MCU_BUS_CLOCK_HZ = 0`). *MCF5407UM/D* permits PSTCLK/BCLKO dividers of 2, 3, or 4 (which at 162.0 MHz core yield 81.0 MHz, 54.0 MHz, or 40.5 MHz).

### Memory Map

The physical address space is decoded via Chip Select registers (CSAR0–CSCR5) and the Module Base Address Register (MBAR):

```
┌───────────────────────┬────────┬────────┬────────────────────────────────┬─────────────────────────────────────────────┐
│ Address Range         │ Size   │ Region │ Authority / Source             │ Description                                 │
├───────────────────────┼────────┼────────┼────────────────────────────────┼─────────────────────────────────────────────┤
│ 0x10000000–0x100003FF │ 1 KiB  │ MBAR   │ Loader (`movec %d0,%mbar`)     │ SIM (System Integration Module registers)   │
│ 0x11000000–0x110007FF │ 2 KiB  │ CS1    │ Firmware (`g_cs1Base`)         │ Host Data Interface (HDI08) to all 8 DSPs   │
│ 0x12000000–0x127FFFFF │ 8 MiB  │ CS2    │ Loader (`CSAR2=$1200`)         │ Firmware & Patch NOR Flash (U37)            │
│ 0x13000000–0x1300000F │ 16 B   │ CS3    │ Loader (`CSAR3=$1300`)         │ Philips ISP1181ADGG USB Controller (U24)    │
│ 0x14000000–0x1400FFFF │ 64 KiB │ CS4    │ Emulation (Configured Default) │ Front Panel Display SRAM Buffer             │
│ 0x15000000–0x1500000F │ 16 B   │ CS5    │ Firmware (`g_cs5Base`)         │ Hardware Latches, Model Straps & Panel P7   │
│ 0x30000000–0x307FFFFF │ 8 MiB  │ SDRAM  │ Firmware (`0x30000400`)        │ Main System SDRAM (RTOS, Heaps, Patches)    │
└───────────────────────┴────────┴────────┴────────────────────────────────┴─────────────────────────────────────────────┘
```

*Note: CS0 is a 512 KiB boot ROM window mapped to `U21`; initial boot begins at the CSBOOT vector.*

### System Integration Module (SIM) & Peripherals

The MCF5407 SIM manages on-chip peripherals and bus arbitration mapped at `MBAR` (`0x10000000`):

| Offset | Register / Peripheral | Description |
|---|---|---|
| `MBAR + 0x006` | `IRQPAR` | Pin Assignment: remaps external interrupt pins (`IRQ1`, `IRQ3`, `IRQ5`, `IRQ7`) |
| `MBAR + 0x04B` | `AVCR` | Autovector Control: configures autovectored interrupt vectors 25–31 |
| `MBAR + 0x04C .. 0x055` | `ICR0 .. ICR9` | Interrupt Control Registers prioritizing 10 internal sources |
| `MBAR + 0x140` | `Timer 1` | System Tick: drives RTOS task scheduling and telemetry loops |
| `MBAR + 0x180` | `Timer 2` | Interval Timer: microsecond delay calibration and timeout enforcement |
| `MBAR + 0x1C0` | `UART0` | Physical MIDI I/O: 31,250 baud (divider `0x0036`), mapped to `ICR4` (vector `0x42`) |
| `MBAR + 0x280` | `M-Bus` | Master I²C interface: drives panel Maxim MAX1039 ADC polling loop |

### ColdFire Exception Model & Stack Frames

The MCF5407 implements the standard ColdFire 8-byte exception frame:

```
+0x00  [ FORMAT 31:28 ] [ FS[3:2] 27:26 ] [ VEC[7:0] 25:18 ] [ FS[1:0] 17:16 ] [ Status Register 15:0 ]
+0x04  [ Program Counter 31:0                                                                           ]
```

- **Self-Aligning Stack**: Frame base aligns to a 4-byte boundary: `base = (SP - 8) & ~3`. Misalignment is recorded in `FORMAT` (`4 + (SP & 3)`), producing formats `$4`, `$5`, `$6`, and `$7`.
- **Fault Status (FS)**: Fault code is split across bits 27:26 and 17:16, distinguishing instruction fetch, operand read, operand write, and write-protect faults.
- **Version 4 Return PC**: On access errors, stacked PC points directly to the faulting instruction (*MCF5407UM/D* folio 4-17), unlike V2/V3 cores which stack mid-instruction.
- **Vector Base Register (VBR)**: VBR must be 1 MByte aligned (`VBR[19:0]` read as zero). The 1,024-byte vector table occupies `VBR + 0x000` through `VBR + 0x3FC`.

## Flash Memory & System SDRAM

The G2 utilizes a partitioned dual-flash architecture separating boot initialization from runtime firmware and patch storage:

- **Boot Flash (`U21`)**: STMicroelectronics or AMD M29LV040B 512 KiB uniform sector Flash on CS0 / CSBOOT. Holds ColdFire reset vector, board diagnostics, SDRAM initialization, and stage-1 bootloader.
- **Patch & OS Flash (`U37`)**: AMD / Spansion Am29LV640DU 8 MiB Flash on CS2 (`0x12000000`–`0x127FFFFF`), decoded with mask register `CSMR2=$007F0001`.

| Flash Offset | Size | Purpose | Contents |
|---|---|---|---|
| `0x000000–0x07FFFF` | 512 KiB | Main RTOS Runtime | `CODE_30000400.bin` image unpacked into SDRAM at `$30000400` |
| `0x080000–0x0FFFFF` | 512 KiB | System Resources | System lookup tables, parameter curves, and UI display data |
| `0x100000–0x1FFFFF` | 1 MiB | DSP Kernel Binaries | DSP bootstrap loaders, resident audio kernels, math routines |
| `0x200000–0x4FFFFF` | 3 MiB | Factory Presets | Factory patch banks (Banks 1–8) and performance setups |
| `0x500000–0x7FFFFF` | 3 MiB | User Storage | User patch banks (Banks 9–16), system settings, MIDI setups |

### System SDRAM Configuration

Main system memory consists of 8 MiB of SDRAM mapped from `0x30000000` through `0x307FFFFF`. During initial boot, the loader configures SDRAM controller registers (`SDCR`, `SDTR`, `SDAR0`, `SDMR0`):

1. Sets CAS latency (2 cycles) and refresh rate intervals.
2. Unpacks the operating system image into base address `0x30000400`.
3. Allocates RTOS execution heaps, USB ring buffers, MIDI queues, dynamic patch graphs, and modulation tables.

## Motorola DSP56367 Octa-DSP Array

Audio synthesis in the G2 is executed across a cluster of up to eight **Motorola DSP56367** processors:

- **DSP 0 .. 3**: Baseboard DSPs (present on G2, G2X, and G2 Engine).
- **DSP 4 .. 7**: Voice Expansion board DSPs (pre-installed on G2X; optional expansion on G2 and G2 Engine).

```
                            ┌────────────────────────────────────────┐
                            │    MCF5407 ColdFire V4 Host (162MHz)   │
                            └───────────────────┬────────────────────┘
                                                │ CS1 Parallel Host Bus (A3..A10)
                 ┌──────────────────────────────┴──────────────────────────────┐
                 │                                                             │
        Baseboard DSPs (0..3)                                        Expansion DSPs (4..7)
  ┌──────────────┬──────────────┐                              ┌──────────────┬──────────────┐
  │    DSP 0     │    DSP 1     │                              │    DSP 4     │    DSP 5     │
  │ Voice Gen /  │ Voice Gen /  │                              │ Voice Gen /  │ Voice Gen /  │
  │ Master Audio │ Audio Path   │                              │ Audio Path   │ Audio Path   │
  └──────┬───────┴──────┬───────┘                              └──────┬───────┴──────┬───────┘
         │              │            ESAI Serial TDM Ring             │              │
         │       ┌──────┴─────────────────────────────────────────────┴───────┐      │
         └───────┤  DSP 2: Voice Gen / FX  │  DSP 3: Voice Gen / FX           ├──────┘
                 │  DSP 6: Voice Gen / FX  │  DSP 7: Voice Gen / FX           │
                 └────────────────────────────────────────────────────────────┘
```

### Array Topology & Processor Allocation

The eight DSPs are organized into functional roles determined dynamically by the loaded performance:

- **DSP 0 (Master Coordinator)**: Interfaces directly with audio converters via ESAI. It collects submixes from all other DSPs, applies master equalization and gain, and outputs the final 4-channel audio streams.
- **DSPs 1–3**: Baseboard voice synthesis engines and insert effects processors (reverb, delay, chorus, vocoder).
- **DSPs 4–7**: Expansion voice synthesis engines, dynamically doubling polyphony when fitted.

### DSP56367 Internal Memory Architecture

According to *DSP56367 User's Manual (DSP56367UM)* section 3.1:

- **Program (P) Memory**: 3K × 24 internal RAM (1K usable as instruction cache or ROM patch), 40K × 24 internal ROM, and 192 × 24 bootstrap ROM (`$FF0000`–`$FF00BF`).
- **Data (X) Memory**: 13K × 24 internal RAM, 32K × 24 internal ROM.
- **Data (Y) Memory**: 7K × 24 internal RAM, 8K × 24 internal ROM.
- **Memory-Switch Mode**: Internal logic can reallocate 5K from X RAM and 2K from Y RAM into P space, expanding executable Program RAM up to 10K × 24 words.

### Clocking & Frame Calculations

- **DSP Core Clock**: 147,456,000 Hz derived from a crystal oscillator.
- **Audio Sample Rate**: Exactly 96,000 Hz native audio rate.
- **Cycles per Frame**:
  $$\frac{147,456,000 \text{ Hz}}{96,000 \text{ Hz}} = 1,536 \text{ DSP clock cycles per audio frame}$$
- **Control Pass Timing**: Modulations, envelopes, LFOs, and sequencer logic update at 24,000 Hz (1 control frame every 4 audio frames):
  $$4 \times 1,536 = 6,144 \text{ DSP clock cycles per control block}$$

### Dynamic Patch Graph Execution vs Fixed-Architecture Synths

Fixed-architecture synths execute a static ROM binary with fixed DSP instructions. The G2 operates on a dynamic paradigm:

1. The user builds or edits a modular patch in the G2 Editor software.
2. The editor compiles the modular wiring graph into a linear chain of optimized DSP assembly routines.
3. The ColdFire MCU receives the compiled patch stream over USB and unpacks the 24-bit DSP machine code.
4. The host downloads module code directly into the DSPs' Program RAM (PRAM) via the HDI08 interface.
5. Each DSP executes its designated module chain every audio frame (1,536 cycles), computing oscillators, filters, mixers, and modulators.
6. Voices are allocated dynamically across the DSP array based on cycle budget requirements and polyphony demands.

## Host Interface (HDI08) & CS1 Bus Architecture

Communication between the ColdFire microcontroller and the DSP array takes place over an 8-bit parallel bus driven by Chip Select 1 (`CS1` at `0x11000000`).

### Active-Low One-Cold Addressing

Instead of using binary address decoding logic, the G2 routes address lines **A3 through A10** directly as individual, active-low chip-select enables to the HDI08 ports of DSP 0 through DSP 7:

| Address Bit | Active-Low Selection Mask | Selected Target |
|---|---|---|
| **A3** = 0 | `0x110007F0` | **DSP 0** selected (Baseboard) |
| **A4** = 0 | `0x110007E8` | **DSP 1** selected (Baseboard) |
| **A5** = 0 | `0x110007D8` | **DSP 2** selected (Baseboard) |
| **A6** = 0 | `0x110007B8` | **DSP 3** selected (Baseboard) |
| **A7** = 0 | `0x11000778` | **DSP 4** selected (Expansion) |
| **A8** = 0 | `0x110006F8` | **DSP 5** selected (Expansion) |
| **A9** = 0 | `0x110005F8` | **DSP 6** selected (Expansion) |
| **A10** = 0 | `0x110003F8` | **DSP 7** selected (Expansion) |

- **Deselect All (Idle)**: Offset `0x7F8` (`11111111000`b in bits A10..A3) pulls all select lines HIGH, preventing bus contention.
- **Broadcast Mode**: Offset `0x000` (`00000000000`b in bits A10..A3) asserts ALL select lines LOW simultaneously. Used by the host to write identical configuration blocks, tempo sync triggers, and boot code to all DSPs in a single transaction.

### HDI08 Register Map & Bus Protocols

According to *DSP56367UM* Table 8-8 (section 8.6, folio 8-19), lines A2..A0 select the internal HDI08 registers from the host side:

| Offset | Register | Direction | Function |
|---|---|---|---|
| `+0x00` | `ICR` | Read / Write | Interface Control Register (bit 5 `HLEND` selects byte endianness) |
| `+0x01` | `CVR` | Read / Write | Command Vector Register (host command triggers, vector numbers) |
| `+0x02` | `ISR` | Read Only | Interface Status Register (RXDF, TXDE, TRDY, host flags HF2/HF3) |
| `+0x03` | `IVR` | Read / Write | Interrupt Vector Register |
| `+0x04` | Reserved | Read Only | Reserved (reads `0x00`) |
| `+0x05` | Data High / Low | Read / Write | `RXH / TXH` (`HLEND=0`) or `RXL / TXL` (`HLEND=1`) |
| `+0x06` | Data Mid | Read / Write | `RXM / TXM` (bits 15:8) |
| `+0x07` | Data Low / High | Read / Write | `RXL / TXL` (`HLEND=0`) or `RXH / TXH` (`HLEND=1`) |

Byte lane ordering is governed by the host-writable `HLEND` bit (`ICR` bit 5). When `HLEND=0` (default big-endian), byte `+0x05` transfers bits 23:16 and byte `+0x07` transfers bits 7:0. Accessing `RXL` (offset `+0x07` with `HLEND=0`) latches the 24-bit word and toggles the transfer handshake flags (`TXDE` / `RXDF`).

## DSP Boot Sequence

The Nord Modular G2 executes a four-phase bootstrap sequence to bring the octa-DSP cluster from hardware reset to real-time audio synthesis:

### Phase 1: Hardware Bootstrap

1. The ColdFire MCU asserts the shared DSP hardware reset line via the control latch at CS5 (`g_cs5Base`).
2. DSP hardware boot mode pins (MODA–MODD) are strapped for Mode C (HDI08 Bootstrap mode; *DSP56367UM* section 6.3).
3. Upon reset release, each DSP56367 executes its on-chip Bootstrap ROM at `0xFFFF00` (192 words).
4. As defined in Appendix A.1 (`HDI08CONT`), the bootstrap ROM polls `HRDF` in `HSR` and reads `HORX`: receiving a word count, a P-space start address (`P:0x0000`), and N program words without requiring any CVR access. (The ROM assembly checks `HF0` only, despite manual comments mentioning `HF1`).

### Phase 2: Host Stream Protocol

1. The ColdFire MCU accesses CS1 using **Broadcast Mode** (offset `0x000`), asserting all eight DSP select lines simultaneously.
2. The host streams the resident bootstrap program (192 words) into DSP Program Memory starting at `P:0x0000`.
3. Because Broadcast Mode is active, all populated DSPs receive and execute the identical base kernel concurrently.
4. Each DSP initializes its internal PLL multiplier to establish the 147.456 MHz core clock and prepares communication buffers.

### Phase 3: PRAM Code Injection

1. The host parses the patch graph received from the editor or stored in Flash.
2. The host addresses each DSP individually via its unique active-low address (A3..A10).
3. The host injects compiled 24-bit module routines directly into PRAM (voice synthesis routines to voice DSPs, effects algorithms to DSP 0 and DSP 2).
4. Filter coefficients, wavetable tables, and modulation matrices are streamed into DSP Data Memory (X and Y RAM).

### Phase 4: Patch Execution & Frame Synchronization

1. The host issues an execution start trigger by writing to the Command Vector Register (`CVR` at `+0x01`).
2. The DSP interrupt vector branches to the patch execution loop at `P:0x0100`.
3. The DSP locks into synchronous 96 kHz frame execution, synchronized by ESAI audio clock interrupts:

```
      ColdFire Host (MCF5407)                     DSP Array (DSP56367)
                │                                         │
                │─── Assert Reset via CS5 ───────────────→│ Reset held low
                │─── Release Reset (HDI08 Boot Mode) ────→│ Bootstrap ROM active (0xFFFF00)
                │                                         │
                │─── Broadcast 192 words to P:0x0000 ────→│ Load resident kernel (all DSPs)
                │                                         │ Initialize PLL to 147.456 MHz
                │                                         │
                │─── Individual CS1 Writes (A3..A10) ────→│ Inject compiled PRAM modules
                │─── Stream Modulation Tables (X/Y) ─────→│ Populate waveform/filter tables
                │                                         │
                │─── Trigger Execution Vector (CVR) ─────→│ Jump to P:0x0100
                │                                         │ ── Running 96 kHz Frame Loop ──
```

## Inter-DSP Audio Routing (ESAI Serial Bus)

Audio signals and voice submixes pass between DSPs and physical DACs/ADCs via the **Enhanced Serial Audio Interface (ESAI)** operating in Time Division Multiplexing (TDM) network mode at a 96,000 Hz frame rate across 4 balanced inputs and 4 balanced outputs:

- **Dual ESAI Interfaces**: The DSP56367 integrates two Enhanced Serial Audio Interfaces (*DSP56367UM* Chapter 10). ESAI resides in X memory space, while ESAI_1 is mapped into Y memory space (`TSMA_1` at `Y:$FFFF99`). The two interfaces share four data pins, with ESAI_1 omitting `HCKR`/`HCKT`.
- **Emulation Substrate Note**: Upstream Gearmulator configures `Peripherals56311` for DSP emulation. However, the physical DSP56311 contains only ESSI (with no ESAI support), whereas the G2 relies on the DSP56367's true 32-slot ESAI TDM network mode.

### Audio Bus Timing & Inter-DSP Multiplexing

The high native sample rate (96 kHz) requires precise bit clock and frame sync derivation across the DSP cluster:

| Parameter | Value |
|---|---|
| **Audio Sample Rate** | 96,000 Hz |
| **ESAI Frame Rate** | 96,000 frames/second |
| **DSP Core Clock** | 147,456,000 Hz |
| **Cycles per Audio Frame** | 1,536 DSP clock cycles |
| **Words per TDM Frame** | Up to 32 time slots per serial line |
| **Word Length** | 24 bits fixed-point |
| **Inter-DSP Bandwidth** | > 73.7 Mbit/s aggregate serial throughput |

Voice DSPs output audio streams into assigned TDM time slots on the shared serial ring. DSP 0 reads these submixes from the serial bus without CPU intervention, mixes voices into output buses, applies master effects, and streams the final stereo master to physical DACs.

## Front Panel & User Interface Subsystem

The physical user interface of the G2 keyboard models is managed by the ColdFire MCU via dedicated chip-select windows and serial buses:

- **5 High-Contrast LCD Displays**: 1 Main Patch LCD (left) displaying patch name and slot status, plus 4 multi-cell parameter displays (right) positioned above the encoder columns, showing module names, rotary values, and parameter status.
- **8 Endless Rotary Encoders**: Surrounded by circular rings of 15 red LEDs that illuminate dynamically to indicate parameter position, bipolar offset, or modulation depth.
- **Dedicated Panel SRAM (CS4)**: In emulation, 64 KiB of static RAM mapped at `0x14000000` buffers display bitmaps and LED states. The ColdFire updates this buffer; panel hardware rasterizes to the displays.
- **Hardware Latches & Straps (CS5)**: Chip Select 5 (`CS5` at `0x15000000`) provides access to physical latches and board identification straps distinguishing G2 Keyboard, G2X, and G2 Engine hardware configurations. Signals route across 26-pin ribbon connector `P7` (Mainboard) to `P1` (Panel), carrying `CS5`, `A0..A2`, `R/W`, `D24..D31`, and I²C lines.

### Maxim MAX1039 I²C Analogue Converter

An onboard Maxim MAX1039 8-bit, 12-channel A/D converter (`U12` on the panel board, 7-bit I²C address `0x65`) is polled continuously over the ColdFire on-chip M-Bus interface. The hardware pins provide 11 settable analogue inputs (`AIN0`–`AIN10`), while pin 13 is `AIN11/REF` tied to reference voltage. The firmware scanning loop monitors five active physical analogue controls:

| Channel Index | Physical Control | Description |
|---|---|---|
| **0** | Master Volume | Master output volume potentiometer |
| **1** | Control Pedal | Rear-panel continuous expression pedal input |
| **2** | Channel Aftertouch | Keybed pressure sensor strip |
| **3** | Pitch Stick | Nord patented wooden strain-gauge stick (dead-band-free) |
| **4** | Modulation Wheel | Front-panel stone modulation wheel (single wheel on G2) |
| **5–10** | Unused / Grounded | Tied directly to ground plane on the panel PCB |

## MIDI & USB Subsystems

The G2 features dual communication interfaces: legacy 5-pin DIN MIDI for musical performance and high-speed USB for real-time editor communication and bulk data exchange.

### DUART MIDI & Sysex Protocol

Physical MIDI I/O is managed by the MCF5407 on-chip DUART Channel A (`UART0`) mapped at `MBAR + 0x1C0`:

- Configured for standard MIDI baud rate: 31,250 baud (divider `0x0036`).
- Interrupt-driven transfers via internal interrupt controller register `ICR4` (autovector vector `0x42`).
- Handles real-time note messages, MIDI CC parameter modulation, and system exclusive patch dumps.

| Byte Offset | Value | Field Description |
|---|---|---|
| **0** | `$F0` | System Exclusive Start |
| **1** | `$33` | Clavia Manufacturer ID |
| **2** | Device ID | Synth Device ID (default: `$00`, broadcast: `$7F`) |
| **3** | `$0B` / `$0C` | Model ID (`$0B` = Nord Modular G2, `$0C` = Nord Modular G2X) |
| **4** | Message Type | Command byte (`$01` = Patch Dump, `$02` = Param Edit, `$10` = Handshake) |
| **5** | Slot Index | Target slot (`$00`–`$03` for Slots A–D, `$04` for Global) |
| **6..N-1** | Data | 7-bit packed parameter or patch payload |
| **N** | `$F7` | System Exclusive End |

### Philips ISP1181ADGG USB Controller & Patch Container

High-speed communication with the PC/Mac G2 Editor is provided by an on-chip Philips ISP1181ADGG USB Full-Speed (12 Mbit/s) peripheral controller (`U24` on the mainboard) mapped at CS3 (`0x13000000`). The interrupt line connects to ColdFire external interrupt `IRQ3` (level 3, autovectored to vector 27):

| Endpoint | Direction | FIFO Size | Purpose |
|---|---|---|---|
| **Endpoint 0** | IN / OUT | 64 bytes | Standard USB enumeration, descriptors, vendor requests |
| **Endpoint 1** | IN | 16 bytes | Real-time panel telemetry, encoder turns, button events |
| **Endpoint 3** | OUT | 64 bytes | Patch graph downloads, DSP machine code, parameter updates |
| **Endpoint 3** | IN | 64 bytes | Real-time VU metering telemetry, patch uploads to editor |

Nord Modular G2 patch files (`.pch2`) are hierarchical structured binary containers using 4-byte ASCII chunk tags and 32-bit big-endian length headers:

| Chunk Tag | Description | Contents |
|---|---|---|
| **`PCH2`** | File Header | Format version identifier and integrity checksums |
| **`HEAD`** | Patch Metadata | Patch title, author, category, timestamp, comments |
| **`MODG`** | Module Graph | List of audio and control modules, types, grid coordinates |
| **`CONN`** | Cable Connections | Wiring list connecting module output jacks to module input jacks |
| **`PARM`** | Parameter Table | Initial knob positions, switch states, bipolar attenuators |
| **`MRPH`** | Morph Assignments | Modulation routing from physical controllers to parameter targets |
| **`CODE`** | DSP Bytecode | Precompiled 24-bit DSP machine code blocks for PRAM injection |

## Cycle-Accurate Emulation Architecture (rg2Lib)

The Red Gecko 2 implementation in Gearmulator is housed under `source/claudia/rg2/rg2Lib/` and consists of two cleanly decoupled architectural layers:

### Clean-Room ColdFire Execution Engine (mcf5407)

- **Independent Core**: Clean-room ColdFire V4 execution engine written from scratch in Nim and transpiled to portable C11 under the MIT License.
- **GPL Cleanliness**: Contains zero lines of code, headers, or algorithms from Musashi or upstream `mc68k`.
- **Instruction Support**: Complete coverage of ColdFire ISA_A arithmetic, 32-bit hardware integer divide and multiply, system control registers, and self-aligning 8-byte format exception frames.

### Substrate Architecture & Multi-Rate Scheduler

The C++ hardware substrate in `rg2Lib` models the physical board assembly:

- `Board` (`board.cpp`): Bridges ColdFire context with memory-mapped devices, chip-select decoders, and the multi-DSP scheduler.
- `Sim` (`sim.cpp`): Models the MCF5407 System Integration Module, chip select registers, and timers.
- `InterruptController` (`interruptController.cpp`): Implements the two-tier interrupt arbiter, prioritizing internal peripheral sources and external pins.
- `Hdi08Adapter` (`hdi08Adapter.cpp`): Emulates active-low one-cold address decode (A3..A10) and bidirectional 24-bit FIFO handshakes.
- `Scheduler` (`scheduler.cpp`): Coordinates the 162.0 MHz ColdFire CPU and eight 147.456 MHz DSP56367 processors using rational cycle-debt accounting, budgeting instruction cycles against the 96 kHz audio frame period (1,536 DSP cycles).

### Emulation Status & Known Limitations

- **Resident Kernel Audio Verified**: Booting and baseline audio synthesis execute cleanly through resident firmware kernel (`t1_boot` and audio test suites pass).
- **DSP JIT Parameter Engine**: User patch compilation is constrained by an open issue in the DSP JIT engine: DO-loop end addresses are not yet re-derived when the guest writes to Loop Address (`LA`) via the host port. Audio output in current builds originates from the resident firmware kernel.

## Reverse Engineering Notes

All technical specifications documented above were uncovered through reverse-engineering the physical hardware, schematics, and binary firmware images.

### Clock Oscillator & Core Frequency Ambiguities

During hardware inspection of the G2 mainboard, an intriguing discrepancy was uncovered regarding system clocking:

1. The physical crystal oscillator can on the PCB is stamped **53.620 MHz**.
2. The Clavia factory schematics label the corresponding clock net entering the MCU as `CPU Ck 54MHz`.
3. The catalog speed grade rating for the processor is 162.0 MHz (`MCF5407CAI162`).

If the internal PLL applies an integer 3× multiplication factor to the 53.620 MHz oscillator, the true core clock frequency is 160.860 MHz, representing a 0.7% deviation from nominal 162.0 MHz. In the emulator, 162.0 MHz is currently modeled. Furthermore, the MCU bus clock divider remains unmeasured on physical hardware; consequently, `G2_MCU_BUS_CLOCK_HZ` is set to 0 (underived) in `rg2Lib`.

### Board Namespaces & Component Designation Recovery

A frequent source of initial confusion when studying the G2 schematics was the presence of duplicate component designators. Reverse engineering revealed that Clavia maintained independent component namespaces for each PCB assembly:

- `ModularG2_MainBoard`: Contains MCU, DSPs 0–3, Flash, SDRAM, USB, and audio converters.
- `ModularG2_Panel`: Contains display latch logic, encoders, buttons, and the MAX1039 ADC.

Both boards carry components labeled `U1` through `U17`. For instance, `MainBoard.U6` is a digital logic IC on the mainboard, while `Panel.U6` is a `74AC138` 3-to-8 decoder driving display strobes.

### Mainboard U18 and U25 Identification

Early third-party notes hypothesized that `U18` was a power regulation IC and `U25` was DSP 2. Detailed tracing of the schematic netlists and physical PCB traces conclusively proved:

- **`U18` is DSP 2** (`DSP56367` baseboard processor).
- **`U25` is an LMS8117ADT-1.8** linear voltage regulator providing the dedicated 1.8V core rail to the DSP array.
- High-speed DSP SRAM is provided by `U8` through `U11` (`AS7C34098-12TC`, four 256K×16 fast SRAM chips).

### Front Panel Ribbon P7 Bus Trace

Tracing the 26-pin ribbon cable connecting `MainBoard.P7` to `Panel.P1` resolved how the front panel is mapped into the ColdFire memory map:

- **Pin 13**: Carries **`CS5`** (Chip Select 5), enabling front panel latch and strap reads.
- **Pins 14–16**: Carry address lines `A0..A2` for selecting panel registers.
- **Pin 12**: Carries the `R/W` control strobe.
- **Pins 18–25**: Carry the 8-bit data bus `D24..D31`.
- **Pins 9–10**: Carry the I²C `SCL` and `SDA` signals from the ColdFire M-Bus controller to the MAX1039 ADC.
- **CS4 Display Staging Buffer**: In emulation, Chip Select 4 (`CS4`) defaults to a 64 KiB staging window at `0x14000000` buffering front panel display data, pending authoritative GDB trace confirmation of physical board routing.

### Peripheral Protocol Recovery (ISP1181 & MAX1039)

- **Philips ISP1181ADGG USB Controller**: Because standalone technical manuals for the legacy ISP1181 were unavailable in vendor archives, the register programming model in `mcf5407` was derived by referencing the *Philips ISP1362 Single-Chip Universal Serial Bus On-The-Go Controller* specification (Rev. 06) and analyzing USB packet captures from the running hardware.
- **Maxim MAX1039 ADC**: Analysis of the ColdFire firmware scanning routine revealed that of the 12 analogue channels provided by the MAX1039, exactly five are active physical controls (*Maxim MAX1036–MAX1039 Low-Power 8-Bit ADCs*, document 19-3255). Channels 5 through 10 are tied to the PCB ground plane, and pin 13 is wired as the reference voltage (`AIN11/REF`).

## Sources & References

1. **Motorola Inc.**, *MCF5407 ColdFire® Integrated Microprocessor User's Manual*, Order Number `MCF5407UM/D`, Rev. 0.1, November 2001.
2. **Freescale Semiconductor**, *DSP56367 24-Bit Digital Signal Processor User's Manual*, Document Number `DSP56367UM`, Rev. 2.1, August 2006.
3. **Freescale Semiconductor**, *ColdFire® Family Programmer's Reference Manual*, Rev. 3 (`CFPRM`), March 2005.
4. **Motorola Inc.**, *MCF5485 ColdFire Microprocessor Hardware Specification*, Document `MCF5485EC`, Rev. 4, December 2004.
5. **Philips Semiconductors**, *ISP1362 Single-Chip Universal Serial Bus On-The-Go Controller*, Data Sheet Rev. 06, 2004.
6. **Maxim Integrated Products**, *MAX1036–MAX1039 Low-Power, 8-Bit, Multichannel ADCs with I²C Compatible Interface*, Document `19-3255`, Rev. 4, October 2005.
7. **Clavia DMI AB**, *Nord Modular G2 / G2X Schematics and Hardware PCB Assemblies*, 2003–2004.
