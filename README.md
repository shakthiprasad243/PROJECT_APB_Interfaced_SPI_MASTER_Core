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
