# APB-Interfaced SPI Master Core
### Synthesizable AMBA APB3 to Full-Duplex SPI Controller RTL Design

[![Verilog](https://img.shields.io/badge/Language-Verilog_2001-blue.svg)](#)
[![Bus Protocol](https://img.shields.io/badge/Interface-AMBA_APB3-orange.svg)](#)
[![Serial Protocol](https://img.shields.io/badge/Interface-SPI_Full--Duplex-brightgreen.svg)](#)
[![Design Stage](https://img.shields.io/badge/Stage-RTL_Simulation_Synthesis-success.svg)](#)

A fully synthesizable **AMBA APB3 to Serial Peripheral Interface (SPI)** bridge controller implemented in Verilog RTL. This IP core acts as an APB3 slave on the internal system-on-chip (SoC) bus and an SPI master/peripheral interface on the external boundary, allowing embedded processors to configure serial transfers, tune baud rates, and execute high-speed, full-duplex data streams.

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

The **APB-Interfaced SPI Master Core** bridges an on-chip register access bus to off-chip synchronous serial peripherals.

```text
                     AMBA APB3 BUS
                           │
                 ┌─────────▼──────────┐
                 │   SPI CONTROLLER   │
                 │ ┌────────────────┐ │
                 │ │ APB Slave      │ │
                 │ │ Interface      │ │
                 │ └───┬────────┬───┘ │
                 │     │        │     │
                 │ ┌───▼───┐┌───▼───┐ │
                 │ │ Baud  ││ Slave │ │
                 │ │ Rate  ││ Select│ │
                 │ │ Gen   ││ Gen   │ │
                 │ └───┬───┘└───┬───┘ │
                 │     │        │     │
                 │ ┌───▼────────▼───┐ │
                 │ │  SPI Shifter   │ │
                 │ └───────┬────────┘ │
                 └─────────┼──────────┘
                           │
                ┌──────────┼──────────┐
                │          │          │
              SCLK        MOSI       SS_n (MISO in)
```
#2. Why APB-Based SPI?

SPI is an unbuffered, unaddressed physical interface that lacks an intrinsic software programming model. Combining it with an AMBA APB3 slave wrapper introduces a memory-mapped programming model:InterfaceProtocolPrimary PurposeHost InterfaceAMBA APB3Memory-mapped register writes, control word configuration, baud rate selection, and interrupt handling.Line InterfaceSPISynchronous, full-duplex serial data transmission to external flash, ADCs, DACs, and sensors.

#3. Overall Architecture
```text
The core partitions logic into four modular, testable RTL blocks:
                              APB MASTER (CPU / DMA)
                                         │
                 ┌───────────────────────┴───────────────────────┐
                 │                                               │
                 ▼                                               │
          ┌──────────────┐                                       │
          │  APB SLAVE   │                                       │
          │  INTERFACE   │                                       │
          └──────┬───────┘                                       │
                 │ Configuration Registers & Transmit Data       │
        ┌────────┴───────────────────────────────────────────────┤
        │                                                        │
        ├────────────────────────────┐                           │
        ▼                            ▼                           │
 ┌─────────────┐              ┌─────────────┐                    │
 │  Baud Rate  │              │Slave Select │                    │
 │  Generator  │              │  Generator  │                    │
 └──────┬──────┘              └──────┬──────┘                    │
        │ SCLK                       │ SS_n                      │
        │ Timing Strobes             │ Transfer In Progress      │
        └─────────────┬──────────────┘                           │
                      │                                          │
                      ▼                                          │
               ┌─────────────┐                                   │
               │ SPI SHIFTER │                                   │
               └──────┬──────┘                                   │
                      │                                          │
         ┌────────────┼────────────┬─────────────┐               │
         ▼            ▼            ▼             ▼               ▼
       MOSI         MISO         SCLK          SS_n            PRDATA
     (Output)      (Input)     (Output)      (Output)         (To APB)
```
APB Slave Interface: Decodes read/write cycles, latches register writes, asserts PREADY, and formats interrupt lines.

Baud Rate Generator: Divides the peripheral clock PCLK to produce the serial shift clock SCLK along with single-cycle sampling/shifting strobes.

Slave Select Generator: Controls active-low target select lines (SS_n), guarantees hold/setup margins, and flags active transfers via tip.

SPI Shifter: Hosts the parallel-load serializer and serial-in deserializer to perform full-duplex transfers according to selected polarity and phase.

#4. Master and Slave Roles

Understanding the protocol boundaries avoids interface confusion:
```text
   +--------------------+               +--------------------+               +--------------------+
   |   HOST PROCESSOR   |   AMBA APB3   |   SPI CONTROLLER   |      SPI      |  EXTERNAL SENSOR   |
   |                    | ------------> |                    | ------------> |                    |
   |    [APB MASTER]    |               |    [APB SLAVE]     |               |    [SPI SLAVE]     |
   |                    |               |    [SPI MASTER]    |               |                    |
   +--------------------+               +--------------------+               +--------------------+
```

On the APB Bus: The host CPU/DMA controller acts as the APB Master. The SPI controller acts as an APB Slave responding to read/write transactions.

On the SPI Bus: The SPI controller functions as the SPI Master driving SCLK, SS_n, and MOSI. The peripheral behaves as the SPI Slave driving MISO.
     
