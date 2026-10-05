---
title: "Red Gecko 2 (Nord Modular G2) Technical Information"
layout: default
permalink: /technical/rg2
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

All technical information discovered on the architecture of the Clavia Nord Modular G2 synthesizer family is documented here. This information was gathered through schematic recovery, board reverse engineering, firmware disassembly, signal analysis of hardware test points, and the development of cycle-accurate emulation in the Gearmulator suite.

---

## Hardware Overview

The Clavia Nord Modular G2 (introduced in 2004) is a second-generation hardware-accelerated modular synthesizer system. Unlike fixed-architecture virtual analog synthesizers (such as the Nord Lead or Access Virus), the Nord Modular G2 executes arbitrary, user-constructed modular synthesis graphs. Patches are designed in a dedicated graphical editor on a PC or Mac, compiled into DSP machine code routines, and transferred in real time over USB to the hardware.

The system combines a high-performance **Motorola MCF5407 ColdFire V4 32-bit microcontroller** with an array of up to **eight Motorola DSP56367 24-bit digital signal processors**. The ColdFire MCU acts as the central conductor: managing high-speed USB communication, MIDI I/O, four graphical LCDs, rotary encoders with LED rings, preset storage, and real-time distribution of patch graphs and modulation tables to the DSP array via an 8-channel parallel host bus.

### Hardware Variants

The Nord Modular G2 family was released in three hardware configurations sharing an identical core system architecture:

- **G2 (Keyboard)**: 3-octave keyboard, 4 high-contrast graphical LCD displays, 8 rotary encoders with 15-LED position rings, pitch stick, modulation wheel, baseboard with 4× DSP56367 processors (expandable to 8).
- **G2X (Keyboard)**: 5-octave keyboard with aftertouch, dual modulation wheels, pitch stick, 4 graphical LCDs, factory-fitted voice expansion board with 4 additional DSPs (8× DSP56367 total).
- **G2 Engine (1U Rack)**: Compact rackmount version without the front control panel, LCDs, or keyboard. Fully controllable via USB and MIDI, containing the baseboard 4× DSP56367 (expandable to 8).

### Key Specifications

| Subsystem | Specification |
|---|---|
| **Host Microcontroller** | Motorola MCF5407CAI162 (ColdFire V4, 32-bit) |
| **MCU Core Clock** | 162.0 MHz (54.0 MHz CLKIN, 3× internal PLL multiplier) |
| **DSP Array** | Up to 8× Motorola DSP56367 (24-bit fixed-point, 56300 family) |
| **DSP Core Clock** | 147.456 MHz |
| **Audio Sample Rate** | 96,000 Hz (96 kHz native frame rate) |
| **DSP Clock Cycles per Frame** | Exactly 1,536 DSP clock cycles per audio sample frame |
| **Control Frame Rate** | 24,000 Hz (1 control frame every 4 audio frames; 6,144 DSP cycles) |
| **Audio I/O** | 4 physical balanced inputs, 4 physical balanced outputs (24-bit / 96 kHz) |
| **Host ↔ DSP Interface** | Chip Select CS1, 8-bit parallel HDI08 bus with active-low one-cold chip selects |
| **System SDRAM** | 32 MB (2× 16-bit SDRAM banks, 100 MHz bus) |
| **Firmware Flash** | 4 MB (AMD/Fujitsu CFI compliant, 16-bit bus) |
| **USB Controller** | Philips ISP1181 Full-Speed (12 Mbps) USB device controller |
| **Panel Displays** | 4× multi-cell graphical LCD displays |
| **Panel Controls** | Endless rotary encoders with 15-LED position indicators, buttons, I²C ADC |

---

## Motorola MCF5407 Microcontroller

The Motorola MCF5407 is a high-integration 32-bit microprocessor based on the ColdFire Version 4 (V4) core architecture. It features a decoupled dual-issue execution pipeline with branch prediction, high-speed single-cycle arithmetic and logic execution, and dedicated hardware integer multiply and divide units.

In the G2 architecture, the MCF5407 runs the real-time operating system that parses `.pch2` patch structures received over USB, generates DSP memory layouts, calculates parameter modulations, and schedules low-latency control frames across the DSP cluster.

### Clock Distribution

- **Input Clock (CLKIN)**: 54.0 MHz crystal oscillator.
- **Core Clock**: 162.0 MHz generated via the MCF5407 on-chip PLL (configured with a 3× multiplier).
- **Bus Clock**: 54.0 MHz synchronous internal peripheral and bus clock.

### Memory Map

The MCF5407 decodes its physical address space through dedicated Chip Select Address Registers (CSAR0–CSCR5) and the internal Module Base Address Register (MBAR):

| Address Range | Size | Chip Select | Description |
|---|---|---|---|
| `0x00000000 - 0x01FFFFFF` | 32 MB | SDRAM (DCS0/1) | Main System SDRAM (RTOS, heap, patch buffers, modulation data) |
| `0x10000000 - 0x100003FF` | 1 KB | MBAR (Internal) | SIM (System Integration Module, internal peripheral registers) |
| `0x11000000 - 0x110007FF` | 2 KB | CS1 | Host Data Interface (HDI08) bridge to all 8 DSP56367 processors |
| `0x12000000 - 0x123FFFFF` | 4 MB | CS2 | Firmware Flash Memory (AMD/Fujitsu CFI 16-bit NOR Flash) |
| `0x13000000 - 0x1300000F` | 16 B | CS3 | Philips ISP1181 USB Full-Speed device controller |
| `0x14000000 - 0x1400FFFF` | 64 KB | CS4 | Static RAM for front panel display buffers and encoder state |
| `0x15000000 - 0x1500000F` | 16 B | CS5 | Hardware Latches & Configuration Straps (Model ID, Reset Lines) |

### System Integration Module (SIM) & Peripherals

The MCF5407 SIM manages on-chip peripherals and bus arbitration mapped at `MBAR` (`0x10000000`):

- **Centralized 2-Tier Interrupt Controller**:
  - `MBAR + 0x006` (`IRQPAR`): Remaps external interrupt pins (`IRQ1`, `IRQ3`, `IRQ5`, `IRQ7`).
  - `MBAR + 0x04B` (`AVCR`): Autovector Control Register, configuring hardware autovectored interrupt vectors 25–31.
  - `MBAR + 0x04C .. 0x057` (`ICR0 .. ICR9`): 8-bit Interrupt Control Registers prioritizing 10 internal sources (Software Watchdog, Timers 1/2, M-Bus/I²C, UART0/1, DMA 0–3).
- **General Purpose Timers**:
  - **Timer 1** (`MBAR + 0x140`): Configured as the real-time system tick generator, driving regular task scheduling and telemetry loops.
  - **Timer 2** (`MBAR + 0x180`): High-precision interval timer used for microsecond delay calibration and bus timeouts.
- **UART0 (DUART Channel A)**:
  - Base address `MBAR + 0x1C0`, mapped to internal interrupt source `ICR4` with user vector `0x42` (decimal 66).
  - Configured for 8 data bits, no parity, 1 stop bit (8N1) at a standard MIDI baud rate of 31,250 baud (divider `0x0036`).
  - Transmit buffer (`UTB`) delivers physical MIDI OUT bytes; Receive buffer (`URB`) accepts physical MIDI IN data.
- **M-Bus (I²C Bus Controller)**:
  - Base address `MBAR + 0x280`, controlling the onboard I²C bus connected to the front-panel MAX1039 ADC.

### Exception Handling & ColdFire Vector Model

The MCF5407 implements the standard ColdFire 8-byte, 2-longword exception frame format rather than the traditional 68000 variable-length frame:

```
+0x00  [ FORMAT 31:28 ] [ FS[3:2] 27:26 ] [ VEC[7:0] 25:18 ] [ FS[1:0] 17:16 ] [ Status Register 15:0 ]
+0x04  [ Program Counter 31:0                                                                           ]
```

- **Self-Aligning Stack**: The exception stack frame base is aligned to a 4-byte boundary: `base = (SP - 8) & ~3`. Misalignment is recorded in the 4-bit `FORMAT` field (`4 + (SP & 3)`), generating format types `$4`, `$5`, `$6`, and `$7`.
- **Fault Status (FS)**: The 4-bit fault status code is split across bits 27:26 and 17:16, distinguishing instruction fetch errors, operand read errors, operand write faults, and write-protect violations.
- **Version 4 Return PC Semantics**: On access errors, the stacked PC points directly to the faulting instruction rather than mid-instruction.
- **Vector Base Register (VBR)**: ColdFire V4 requires VBR to be 1 MByte aligned (`VBR[19:0]` are unimplemented in hardware and read as zero). The 1024-byte vector table occupies `VBR + 0x000` through `VBR + 0x3FC`.

---

## Motorola DSP56367 Octa-DSP Array

Audio synthesis in the Nord Modular G2 is executed across a homogeneous cluster of up to eight **Motorola DSP56367** processors:

- **DSP 0 .. 3**: Baseboard DSPs (always present on G2, G2X, and G2 Engine).
- **DSP 4 .. 7**: Voice Expansion board DSPs (pre-installed on G2X; optional expansion daughtercard on G2 and G2 Engine).

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

### Clocking & Frame Calculations

- **Core Clock**: 147.456 MHz crystal-controlled DSP clock.
- **Native Frame Rate**: 96,000 Hz.
- **Cycles per Frame**:
  $$\frac{147,456,000 \text{ Hz}}{96,000 \text{ Hz}} = 1,536 \text{ DSP clock cycles per audio frame}$$
- **Control Pass Timing**: Modulations, LFOs, and envelope computations operate at 24 kHz (one control pass every 4 audio frames):
  $$4 \times 1,536 = 6,144 \text{ DSP clock cycles per control block}$$

### Dynamic Patch Architecture vs Fixed-Architecture Synths

Fixed-architecture synthesizers (such as the Access Virus or Waldorf microQ) run a static ROM image on their DSPs with predefined oscillator, filter, and modulation routines.

The Nord Modular G2 operates completely differently:
1. When a user creates or edits a patch in the G2 Editor, the editor compiles the modular wiring diagram into an optimized, linear list of DSP module routines.
2. The ColdFire MCU receives the compiled patch stream over USB and unpacks the DSP machine code.
3. The host downloads the module code directly into the DSPs' internal 24-bit Program RAM (PRAM) via the HDI08 interface.
4. Each DSP executes its designated module chain every audio frame (1,536 cycles), computing oscillators, filters, mixers, logic gates, and sequencers.
5. Voices are allocated dynamically across the available DSPs based on available execution cycles and voice demand.

---

## Host Interface (HDI08) & CS1 Bus Architecture

Communication between the ColdFire microcontroller and the DSP array takes place over an 8-bit parallel bus driven by Chip Select 1 (`CS1` at `0x11000000`).

### Active-Low One-Cold Chip Select Addressing

Instead of using traditional binary address decoding (which would require a demultiplexer chip), the G2 hardware wires address lines **A3 through A10** directly as individual, active-low chip-select enables to the HDI08 ports of DSP 0 through DSP 7:

| Address Bit | Bit Mask | Selected Target |
|---|---|---|
| **A3** = 0 | `0x110007F0` | **DSP 0** selected |
| **A4** = 0 | `0x110007E8` | **DSP 1** selected |
| **A5** = 0 | `0x110007D8` | **DSP 2** selected |
| **A6** = 0 | `0x110007B8` | **DSP 3** selected |
| **A7** = 0 | `0x11000778` | **DSP 4** selected (Expansion) |
| **A8** = 0 | `0x110006F8` | **DSP 5** selected (Expansion) |
| **A9** = 0 | `0x110005F8` | **DSP 6** selected (Expansion) |
| **A10** = 0 | `0x110003F8` | **DSP 7** selected (Expansion) |

### Idle and Broadcast Modes

- **Deselect All (Idle)**: Address offset `0x7F8` (`11111111000`b in bits A10..A3) pulls all select lines HIGH. No DSP is selected, preventing accidental bus contention.
- **Broadcast Mode**: Address offset `0x000` (`00000000000`b in bits A10..A3) asserts ALL select lines LOW simultaneously.
  - Used by the ColdFire MCU to write identical configuration blocks, global tempo sync triggers, and initial boot code to all DSPs in a single bus transaction.

### HDI08 Register Map

Offsets within each DSP's 8-byte slot (bits A2..A0):

| Offset | Register | Read / Write | Function |
|---|---|---|---|
| `+0x00` | `ICR` | R/W | Interface Control Register (interrupt enables, host flags HF0/HF1) |
| `+0x01` | `CVR` | R/W | Command Vector Register (host command triggers, vector numbers) |
| `+0x02` | `ISR` | Read Only | Interface Status Register (RXDF, TXDE, TRDY, host flags HF2/HF3) |
| `+0x03` | `IVR` | R/W | Interrupt Vector Register |
| `+0x04` | `RXH / TXH` | R/W | Receive / Transmit High Byte (Bits 23:16) |
| `+0x05` | `RXM / TXM` | R/W | Receive / Transmit Mid Byte (Bits 15:8) |
| `+0x06` | `RXL / TXL` | R/W | Receive / Transmit Low Byte (Bits 7:0) |

Writing to `RXL` or reading from `RXL` latches the complete 24-bit word and toggles the transfer handshakes (`TXDE` / `RXDF`).

---

## Inter-DSP Audio Routing (ESAI Serial Bus)

Audio signals pass between DSPs and physical DACs/ADCs via the **Enhanced Serial Audio Interface (ESAI)** operating in Time Division Multiplexing (TDM) mode.

- **Sample Rate**: 96 kHz.
- **Audio Inputs**: 4 channels digitized through 24-bit 96 kHz A/D converters connected to ESAI RX.
- **Audio Outputs**: 4 channels routed to 24-bit 96 kHz D/A converters connected to ESAI TX.
- **Inter-DSP Links**: Voice outputs from individual DSPs are multiplexed across ESAI serial time slots, allowing any voice DSP to stream submixes to the master effects and output DSPs with zero phase jitter.

---

## Peripherals & Front Panel Subsystem

### Philips ISP1181 USB Controller

Communication with the external Mac/PC Nord Modular G2 Editor is provided by an on-chip Philips ISP1181 USB Full-Speed (12 Mbps) device controller mapped at CS3 (`0x13000000`):

- **Control Endpoint 0**: Handles standard USB enumeration, descriptors, and vendor setup requests.
- **Endpoint 3 (Bulk OUT / Bulk IN)**: Configured with a 64-byte FIFO buffer. Carries all real-time patch upload data, parameter tweaks, MIDI over USB, and metering telemetry.
- **Interrupt Line**: Connected to ColdFire external interrupt `IRQ3` (level 3, autovectored to vector 27).

### Front Panel User Interface

The G2 user interface provides instantaneous visual feedback for all modular parameters:

- **4 Graphical LCD Displays**: Each display features a 128×64 dot matrix area divided into dynamic parameter cells, displaying module names, rotary values, filter curves, and sequencer bars.
- **8 Endless Rotary Encoders**: Surrounded by 15-LED circular indicator rings that illuminate dynamically to indicate current parameter position, bipolar offsets, or active modulation depth.
- **Front Panel Static RAM (CS4)**: 64 KB of dedicated fast SRAM at `0x14000000` buffers display bitmaps and LED states, relieving main system SDRAM of display refresh traffic.
- **MAX1039 I²C ADC**: 10-bit, 8-channel analog-to-digital converter polled over the ColdFire M-Bus interface. Converts analog potentials from:
  - Master Volume potentiometer
  - Control Pedal input
  - Keyboard Channel Aftertouch sensor
  - Wooden Pitch Stick (Nord patented strain-gauge mechanism)
  - Modulation Wheel 1 and 2

---

## Cycle-Accurate Emulation in Gearmulator (`rg2Lib`)

The Red Gecko 2 implementation in Gearmulator is housed under `source/claudia/rg2/rg2Lib/` and consists of two cleanly separated architectural layers:

1. **`mcf5407` (MIT License)**:
   - Clean-room, independent ColdFire V4 execution engine written from scratch in Nim and transpiled to portable C11.
   - Has zero lines of code, headers, or algorithms from Musashi or upstream `mc68k`.
   - Complete support for ColdFire ISA_A arithmetic, 32-bit hardware divide/multiply, control registers, and 8-byte format exception frames.
2. **`rg2Lib` Board Substrate (C++)**:
   - `Board` (`board.cpp`): Bridges the ColdFire context with memory-mapped devices, chip-select windows, and the 8-DSP scheduler.
   - `Sim` (`sim.cpp`): Models the MCF5407 System Integration Module, chip select registers, and timers.
   - `InterruptController` (`interruptController.cpp`): Implements the two-tier interrupt arbiter, prioritizing internal sources and external pins.
   - `Hdi08Adapter` (`hdi08Adapter.cpp`): Emulates the active-low one-cold address decode (A3..A10) and bidirectional 24-bit FIFO handshakes.
   - `Scheduler` (`scheduler.cpp`): Implements rational cycle-debt accounting, coordinating the 162.0 MHz ColdFire CPU and eight 147.456 MHz DSP56367 processors into unified, jitter-free 96 kHz audio frames.
