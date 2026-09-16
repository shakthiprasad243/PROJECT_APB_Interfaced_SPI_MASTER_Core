# PROJECT_APB_Interfaced_SPI_MASTER_Core
# APB-Based SPI Controller RTL Design

[![Verilog](https://img.shields.io/badge/Language-Verilog_2001-blue.svg)](#)
[![Bus Protocol](https://img.shields.io/badge/Interface-AMBA_APB3-orange.svg)](#)
[![Serial Protocol](https://img.shields.io/badge/Interface-SPI_Full--Duplex-brightgreen.svg)](#)
[![Design Stage](https://img.shields.io/badge/Stage-RTL_Simulation_Synthesis-success.svg)](#)

A fully synthesizable **AMBA APB3 to SPI (Serial Peripheral Interface)** bridging controller implemented in Verilog RTL. This IP core acts as an APB3 slave on the system bus and an SPI master/peripheral interface externally, enabling processor sub-systems to seamlessly transfer full-duplex serial data with peripheral devices.

---

## Table of Contents
- [1. Project Overview](#1-project-overview)
- [2. Why APB-Based SPI?](#2-why-apb-based-spi)
- [3. Overall Architecture](#3-overall-architecture)
- [4. Master and Slave Roles](#4-master-and-slave-roles)
- [5. APB Side Interface](#5-apb-side-interface)
- [6. SPI Side Interface](#6-spi-side-interface)
- [7. Complete Full-Duplex Data Flow](#7-complete-full-duplex-data-flow)
- [8. Register Map](#8-register-map)
- [9. Register Field Definitions](#9-register-field-definitions)
  - [SPI Control Register 1 (SPI_CR1)](#spi-control-register-1-spi_cr1)
  - [SPI Control Register 2 (SPI_CR2)](#spi-control-register-2-spi_cr2)
  - [SPI Baud Rate Register (SPI_BR)](#spi-baud-rate-register-spi_br)
  - [SPI Status Register (SPI_SR)](#spi-status-register-spi_sr)
  - [SPI Data Register (SPI_DR)](#spi-data-register-spi_dr)
- [10. Operating Modes](#10-operating-modes)
- [11. Submodule Architecture](#11-submodule-architecture)
  - [Block 1: APB Slave Interface](#block-1-apb-slave-interface)
  - [Block 2: Baud Rate Generator](#block-2-baud-rate-generator)
  - [Block 3: SPI Slave Select Generator](#block-3-spi-slave-select-generator)
  - [Block 4: SPI Shifter](#block-4-spi-shifter)
- [12. SPI Modes, Phases, and Ordering](#12-spi-modes-phases-and-ordering)
- [13. Transaction Walkthrough](#13-transaction-walkthrough)
- [14. Interrupts, Errors, and Faults](#14-interrupts-errors-and-faults)
- [15. Key Design Equations](#15-key-design-equations)
- [16. Important Signal Dictionary](#16-important-signal-dictionary)
- [17. Directory Structure](#17-directory-structure)
- [18. Verification, Simulation, and Synthesis](#18-verification-simulation-and-synthesis)
- [19. Scope, Limitations, and References](#19-scope-limitations-and-references)

---

## 1. Project Overview

The **APB-Based SPI Controller** bridges an internal host processor bus to an off-chip SPI bus.

APB BUS
               │
      ┌────────▼─────────┐
      │  SPI CONTROLLER  │
      │ ┌──────────────┐ │
      │ │ APB Slave    │ │
      │ │ Interface    │ │
      │ └──────┬───────┘ │
      │        │         │
      │   ┌────┴────┐    │
      │   ▼         ▼    │
      │  Baud     Slave  │
      │  Rate     Select │
      │  Gen      Gen    │
      │   │         │    │
      │   └────┬────┘    │
      │        ▼         │
      │ ┌──────────────┐ │
      │ │ SPI Shifter  │ │
      │ └──────────────┘ │
      └───────┬─┬─┬──────┘
              │ │ │
         SCLK MOSI MISO SS

 The APB master manages configuration, baud rates, and payloads by reading and writing internal memory-mapped registers. The SPI controller offloads serial-to-parallel translation, clock division, and chip-select line management.

---

## 2. Why APB-Based SPI?

SPI lacks an inherent software programming model or memory bus interface. Integrating an APB slave front-end provides standard CPU read/write access:

| Interface | Protocol | Primary Purpose |
|:---|:---|:---|
| **Host Side** | **AMBA APB3** | Register programming, baud config, transmit/receive FIFO/buffer access, status monitoring |
| **Line Side** | **SPI** | Synchronous, full-duplex serial transmission to external sensors, memories, and codecs |

---

## 3. Overall Architecture

The core partitions logic into four isolated, specialized blocks:
APB MASTER
                         │
         ┌───────────────┴────────────────┐
         ▼                                │
  ┌──────────────┐                        │
  │  APB SLAVE   │                        │
  │  INTERFACE   │                        │
  └──────┬───────┘                        │
         │ Configuration & Data           │
 ┌───────┴────────────────────────────────┤
 │
 ├──────────────────┐
 ▼                  ▼
┌───────────┐     ┌─────────────┐
│ Baud Rate │     │ Slave Select│
│ Generator │     │ Generator   │
└─────┬─────┘     └──────┬──────┘
│ SCLK             │ SS_n
└────────┬─────────┘
▼
┌─────────────┐
│ SPI SHIFTER │
└──────┬──────┘
│
┌────┴────┐
MOSI MISO SCLK SS_n

- **APB Slave Interface:** Decodes bus commands, latches configuration, generates APB transfer responses (`PREADY`), and routes status/interrupts.
- **Baud Rate Generator:** Divides high-frequency `PCLK` to synthesize `SCLK` along with single-cycle sampling/shifting trigger strobes.
- **Slave Select Generator:** Drives active-low chip select (`SS_n`), regulates transaction durations, and exposes the transfer-in-progress (`tip`) flag.
- **SPI Shifter:** Manages shift registers for outbound `MOSI` parallel-to-serial conversion and incoming `MISO` serial-to-parallel assembly.

---

## 4. Master and Slave Roles

| Domain | Bus Master | Bus Slave | Role Description |
|:---|:---|:---|:---|
| **APB Bus** | Host CPU / DMA Controller | **SPI Core (APB Interface)** | CPU controls transfers; SPI core never drives unsolicited reads/writes. |
| **SPI Bus** | **SPI Core** | External Peripheral Device | SPI core drives `SCLK` and asserts `SS_n`; the external device responds on `MISO`. |

---

## 5. APB Side Interface

The APB3 slave macro operates across three sequential bus phases:
[ IDLE ] ──( PSEL=1 )──► [ SETUP ] ──( PENABLE=1 )──► [ ENABLE / ACCESS ] ──► [ IDLE ]

### Signal Manifest
| Signal | Direction | Width | Function |
|:---|:---:|:---:|:---|
| `PCLK` | Input | 1 | APB Bus Clock |
| `PRESETn` | Input | 1 | Asynchronous/Synchronous Active-Low Reset |
| `PSEL` | Input | 1 | Peripheral Select |
| `PENABLE` | Input | 1 | Strobe asserting APB data phase |
| `PWRITE` | Input | 1 | High = Write transaction; Low = Read transaction |
| `PADDR` | Input | 3 | Address bus (`PADDR[2:0]`) indexing registers |
| `PWDATA` | Input | 8 / 32 | Inbound write data from master |
| `PRDATA` | Output | 8 / 32 | Outbound read data to master |
| `PREADY` | Output | 1 | Transfer complete acknowledgment |
| `PSLVERR` | Output | 1 | Transfer error indicator (default tied to `1'b0`) |

---

## 6. SPI Side Interface

| Pin Name | Direction | Active Level | Description |
|:---|:---:|:---:|:---|
| `SCLK` | Output | Configurable | Serial Shift Clock (driven by Master) |
| `MOSI` | Output | Dynamic | Master Out Slave In (Master transmit line) |
| `MISO` | Input | Dynamic | Master In Slave Out (Master receive line) |
| `SS_n` | Output | Active-Low | Peripheral Slave Select line (`0` = Selected) |

---

## 7. Complete Full-Duplex Data Flow

[APB MASTER]                                          [SPI SLAVE]
│                                                     │
│── Write 8'b1010_1100 to SPI_DR ──►                  │
│                                                     │
▼                                                     ▼
[SPI_DR (TX)] ──► [send_data]                              │
│                                   │
├── Assert SS_n = 0 ───────────────►│
├── Start SCLK Generation ─────────►│
│                                   │
[SPI SHIFTER]                             │
MOSI: Serial Output ─────────────────────►│
MISO: Serial Input  ◄─────────────────────│
│                                   │
▼                                   │
[SPI_DR (RX)] ◄── [receive_data]                           │
│                 │                                   │
│                 └── De-assert SS_n = 1 ────────────►│
│                                                     │
│◄── Read Received Byte via APB ──                    │


---

## 8. Register Map

Decoded using a 3-bit internal offset address (`PADDR[2:0]`):

| Offset (`PADDR`) | Register Name | Acronym | Access | Reset Value | Description |
|:---:|:---|:---:|:---:|:---:|:---|
| `3'b000` (`0x0`) | SPI Control Register 1 | `SPI_CR1` | R/W | `8'h04` | Operational enables, Clock polarity & phase |
| `3'b001` (`0x1`) | SPI Control Register 2 | `SPI_CR2` | R/W | `8'h00` | Fault detect enables, Pin modes, Stop/Wait behavior |
| `3'b010` (`0x2`) | SPI Baud Rate Register | `SPI_BR` | R/W | `8'h00` | Clock pre-scaler and rate division select |
| `3'b011` (`0x3`) | SPI Status Register | `SPI_SR` | RO | `8'h20` | Core status flags (TX Empty, RX Done, Fault) |
| `3'b101` (`0x5`) | SPI Data Register | `SPI_DR` | R/W | `8'h00` | Transmit latch (Write) and Receive latch (Read) |

---

## 9. Register Field Definitions

### SPI Control Register 1 (`SPI_CR1`)
7        6       5        4      3      2      1       0
┌──────┬──────┬───────┬──────┬──────┬──────┬──────┬───────┐
│ SPIE │ SPE  │ SPTIE │ MSTR │ CPOL │ CPHA │ SSOE │ LSBFE │
└──────┴──────┴───────┴──────┴──────┴──────┴──────┴───────┘

- **Bit 7 - `SPIE` (SPI Interrupt Enable):** Generates an interrupt if `SPIF` or `MODF` is asserted.
- **Bit 6 - `SPE` (SPI System Enable):** `1` = Core enabled; `0` = Core disabled and held in reset.
- **Bit 5 - `SPTIE` (SPI Transmit Interrupt Enable):** Generates an interrupt when buffer is empty (`SPTEF = 1`).
- **Bit 4 - `MSTR` (Master Mode Select):** `1` = Master Mode (generates SCLK/SS_n); `0` = Slave Mode.
- **Bit 3 - `CPOL` (Clock Polarity):** `0` = Idle low; `1` = Idle high.
- **Bit 2 - `CPHA` (Clock Phase):** `0` = Sample on leading edge; `1` = Sample on trailing edge.
- **Bit 1 - `SSOE` (Slave Select Output Enable):** Enables external automatic assertion of `SS_n`.
- **Bit 0 - `LSBFE` (LSB-First Enable):** `0` = MSB first; `1` = LSB first.

### SPI Control Register 2 (`SPI_CR2`)
7     6     5       4         3        2        1        0
┌─────┬─────┬─────┬────────┬─────────┬───────┬─────────┬──────┐
│  0  │  0  │  0  │ MODFEN │ BIDIROE │   0   │ SPISWAI │ SPC0 │
└─────┴─────┴─────┴────────┴─────────┴───────┴─────────┴──────┘

- **Bit 4 - `MODFEN`:** Mode Fault Enable bit.
- **Bit 3 - `BIDIROE`:** Bidirectional Output Buffer Enable.
- **Bit 1 - `SPISWAI`:** SPI Clocks halt in Low-Power Wait Mode.
- **Bit 0 - `SPC0`:** Pin config switch (Single-wire bidirectional mode selection).

### SPI Baud Rate Register (`SPI_BR`)
7       6       5       4       3       2       1       0
┌─────┬───────┬───────┬───────┬─────┬───────┬───────┬───────┐
│  0  │ SPPR2 │ SPPR1 │ SPPR0 │  0  │ SPR2  │ SPR1  │ SPR0  │
└─────┴───────┴───────┴───────┴─────┴───────┴───────┴───────┘

- **Bits [6:4] - `SPPR[2:0]`:** Prescaler divisor selection bits (1 to 8).
- **Bits [2:0] - `SPR[2:0]`:** Clock division scaling powers ($2^1$ to $2^8$).

### SPI Status Register (`SPI_SR`)
7      6      5       4      3     2     1     0
┌──────┬────┬───────┬──────┬─────┬─────┬─────┬─────┐
│ SPIF │ 0  │ SPTEF │ MODF │  0  │  0  │  0  │  0  │
└──────┴────┴───────┴──────┴─────┴─────┴─────┴─────┘

- **Bit 7 - `SPIF` (SPI Full / Receive Complete Flag):** High when 8 bits have arrived and latched to `SPI_DR`. Cleared by reading `SPI_SR` followed by reading `SPI_DR`.
- **Bit 5 - `SPTEF` (SPI Transmit Buffer Empty):** High when `SPI_DR` is ready for another payload byte.
- **Bit 4 - `MODF` (Mode Fault Flag):** High when an external system conflict pulls master `SS_n` low while `MODFEN=1`.

### SPI Data Register (`SPI_DR`)
- An 8-bit bidirectional address window. 
- **Write:** Targets internal transmission buffer (`TX_BUFFER`).
- **Read:** Returns shadow register containing the most recently gathered frame (`RX_BUFFER`).

---

## 10. Operating Modes

           ┌──────────────┐
           │   SPI_RUN    │ ◄── Core Active (SPE = 1)
           └──────┬───────┘
                  │
    ┌─────────────┴─────────────┐
    ▼                           ▼
┌──────────────┐            ┌──────────────┐
│   SPI_WAIT   │            │   SPI_STOP   │
└──────────────┘            └──────────────┘
(Low power state,           (Core clock gated,
SPISWAI dependent)          Baud generator off)


---

## 11. Submodule Architecture

### Block 1: APB Slave Interface
Decodes input control busses, generates synchronization strobes (`wr_enb = PWRITE & PENABLE & PSEL`, `rd_enb = ~PWRITE & PENABLE & PSEL`), returns low-latency read data (`PRDATA`), and arbitrates the core interrupt line.

### Block 2: Baud Rate Generator
Synthesizes `SCLK` using a synchronous toggle counter driven by `PCLK`. Emits calibrated single-cycle timing pulses (`sclk_rising_edge`, `sclk_falling_edge`) used downstream by the shifter to capture or shift data bits reliably.

### Block 3: SPI Slave Select Generator
Monitors the `send_data` strobe from the APB interface. When asserted, it drives `SS_n = 0`, raises `tip = 1` (Transfer-In-Progress), monitors the bit counter window ($BaudRateDivisor \times 16$), and raises `receive_data` on completion before de-asserting `SS_n = 1`.

### Block 4: SPI Shifter
Contains the primary outbound serializer shift register and inbound deserializer capture register. Based on the configured `CPOL`, `CPHA`, and `LSBFE` bits, it serializes data out via `MOSI` and reads `MISO` on the appropriate clock edges.

---

## 12. SPI Modes, Phases, and Ordering

### Clock Modes Matrix
| Mode | `CPOL` | `CPHA` | Idle `SCLK` | Data Shift Edge | Data Sample Edge |
|:---:|:---:|:---:|:---:|:---:|:---:|
| **Mode 0** | 0 | 0 | Low (`1'b0`) | Falling Edge | Rising Edge |
| **Mode 1** | 0 | 1 | Low (`1'b0`) | Rising Edge | Falling Edge |
| **Mode 2** | 1 | 0 | High (`1'b1`) | Rising Edge | Falling Edge |
| **Mode 3** | 1 | 1 | High (`1'b1`) | Falling Edge | Rising Edge |

### Bit Transmission Order Example (`8'b1010_1100`)
- **MSB-First (`LSBFE = 0`):** `1 ──► 0 ──► 1 ──► 0 ──► 1 ──► 1 ──► 0 ──► 0`
- **LSB-First (`LSBFE = 1`):** `0 ──► 0 ──► 1 ──► 1 ──► 0 ──► 1 ──► 0 ──► 1`

---

## 13. Transaction Walkthrough

PCLK      : ──|‾||‾||‾||‾||‾||‾||‾||‾||‾||‾||‾||‾||‾||‾|
PADDR/DATA: ──<  WRITE SPI_DR (0xA5)  >───────────────────────────────
send_data : ──────|‾‾‾‾|______________________________________________
SS_n      : ───────────____________________/‾‾‾‾‾‾‾‾
SCLK      : ───────────────||‾||‾||‾||‾||‾||‾||‾||‾|__________
MOSI      : ───────────────< Bit 7 >< Bit 6 >...< Bit 0 >─────────────
MISO      : ───────────────< In  7 >< In  6 >...< In  0 >─────────────
tip       : ───────────/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾_
receive_d : ────────────────────────────────────────────|‾‾‾‾|
SPIF      : ─────────────────────────────────────────────────/‾‾‾‾‾‾‾‾


---

## 14. Interrupts, Errors, and Faults

The core provides a single unified active-high interrupt output request line (`spi_interrupt_request`) driven by internal status events:

SPIF  ───┐
├─(OR)──┐
MODF  ───┘       │
├─(AND)──┐
SPIE  ───────────┘        │
├─(OR)──► spi_interrupt_request
SPTEF ───┐                │
├─(AND)──────────┘
SPTIE ───┘


- **Mode Fault (`MODF`):** Triggered if the core is configured as a Master (`MSTR=1`) with Mode Fault enabled (`MODFEN=1`), but its `SS_n` pin is driven low externally, indicating another master is asserting control over the bus.

---

## 15. Key Design Equations

$$\text{Baud Rate Divisor} = (\text{SPPR} + 1) \times 2^{(\text{SPR} + 1)}$$

$$f_{\text{SCLK}} = \frac{f_{\text{PCLK}}}{\text{Baud Rate Divisor}}$$

$$\text{Half Period Counter Limit} = \frac{\text{Baud Rate Divisor}}{2} - 1$$

$$\text{Total Frame Period Counter Duration} = \text{Baud Rate Divisor} \times 16$$

---

## 16. Important Signal Dictionary

├── APB Bus Infrastructure
│   ├── PCLK        : Bus system clock
│   ├── PRESETn     : System-wide active-low reset
│   ├── PSEL        : Select strobe for this target peripheral
│   ├── PENABLE     : Access phase strobe
│   ├── PWRITE      : Transfer direction (1: Write, 0: Read)
│   ├── PADDR[2:0]  : 3-bit Register byte address
│   ├── PWDATA[7:0] : Outbound payload byte from CPU
│   ├── PRDATA[7:0] : Inbound data byte returned to CPU
│   └── PREADY      : Core ready handshake flag
├── SPI Physical Lines
│   ├── SCLK        : Master-driven serial clock
│   ├── MOSI        : Master Out Slave In serialized data
│   ├── MISO        : Master In Slave Out serialized input
│   └── SS_n        : Active-low peripheral chip select
└── Internal Core Strobes
├── send_data   : Single-cycle trigger launching SPI frame
├── receive_data: Strobe marking final bit acquisition
├── tip         : Transfer-in-progress status flag (~SS_n)
└── irq         : Masked top-level interrupt line


---

## 17. Directory Structure

```plaintext
spi_design_lab/
├── rtl/
│   ├── apb_slave.v             # APB3 interface, decoders, CSRs
│   ├── baud_generator.v        # Clock divider & edge strobe generator
│   ├── spi_slave_select.v      # SS_n timing engine and tip tracker
│   ├── spi_shifter.v           # Parallel/Serial bidirectional shift register
│   └── spi_top.v               # Top-level structural wiring wrapper
├── tb/
│   ├── apb_slave_tb.v          # APB interface unit verification
│   ├── baud_generator_tb.v     # Baud rate divider verification
│   ├── spi_slave_select_tb.v   # Chip select duration verification
│   ├── spi_shifter_tb.v        # Serial shifting verification
│   └── spi_top_tb.v            # Full-duplex loopback integration testbench
├── sim/                        # Simulation run scripts and waveforms (.vcd)
├── synth/                      # Synthesis run scripts and gate-level netlists
├── lint/                       # Linter configuration and rule violation logs
└── README.md                   # Project documentation
