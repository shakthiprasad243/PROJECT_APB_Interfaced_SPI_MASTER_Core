# PROJECT_APB_Interfaced_SPI_MASTER_Core
APB-Based SPI Protocol

A synthesizable APB-based Serial Peripheral Interface (SPI) controller implemented in Verilog RTL.

This project combines two protocols:

* AMBA APB3 for communication between a processor/APB master and the SPI controller
* SPI for serial communication between the SPI controller and an external SPI peripheral

The design is divided into four major RTL blocks:

1. APB Slave Interface
2. Baud Rate Generator
3. SPI Slave Select Generator
4. SPI Shifter

The project also includes individual verification environments for the RTL blocks, simulation, linting, and synthesis.

⸻

Table of Contents

* 1. Project Overview
* 2. Why APB-Based SPI?
* 3. Overall Architecture
* 4. Understanding the Master and Slave Roles
* 5. APB Side
* 6. SPI Side
* 7. Complete Data Flow
* 8. SPI Controller Register Map
* 9. SPI Control Register 1
* 10. SPI Control Register 2
* 11. SPI Baud Rate Register
* 12. SPI Status Register
* 13. SPI Data Register
* 14. SPI Operating Modes
* 15. Block 1: APB Slave Interface
* 16. Block 2: Baud Rate Generator
* 17. Block 3: SPI Slave Select Generator
* 18. Block 4: SPI Shifter
* 19. CPOL and CPHA
* 20. MSB-First and LSB-First
* 21. SPI Transaction
* 22. Read Transaction
* 23. Write Transaction
* 24. Transfer Completion
* 25. Interrupt Generation
* 26. Mode Fault Detection
* 27. Reset Behavior
* 28. RTL Block Interaction
* 29. Directory Structure
* 30. Verification
* 31. Simulation
* 32. Lint
* 33. Synthesis
* 34. Key Design Equations
* 35. Important Signals
* 36. Design Limitations and Scope
* 37. Learning Objectives
* 38. References

⸻

1. Project Overview

The APB-Based SPI Protocol project implements an SPI controller that can be configured and controlled through an APB interface.

The basic idea is:

                 APB BUS
                   |
                   |
          +--------v---------+
          |                  |
          |   SPI CONTROLLER |
          |                  |
          | +--------------+ |
          | | APB Slave    | |
          | | Interface    | |
          | +------+-------+ |
          |        |         |
          |   +----+----+    |
          |   |         |    |
          |   v         v    |
          | Baud      Slave  |
          | Rate      Select |
          | Generator Generator
          |   |         |    |
          |   +----+----+    |
          |        |         |
          |        v         |
          |   +----------+   |
          |   | Shifter  |   |
          |   +----------+   |
          |       | | |      |
          +-------|-|-|------+
                  | | |
                 SCLK
                 MOSI
                 MISO
                  SS

The APB master configures the SPI controller by writing to its registers.

The SPI controller then performs the actual serial communication with an external SPI device.

⸻

2. Why APB-Based SPI?

SPI itself does not define how software configures the SPI controller.

A processor needs some internal bus interface to:

* configure SPI
* select master/slave operation
* configure CPOL and CPHA
* configure baud rate
* write transmit data
* read received data
* read status
* enable interrupts

APB provides this control interface.

Therefore:

Processor
    |
    | APB
    v
+-----------+
| SPI Core  |
+-----------+
    |
    | SPI
    v
SPI Peripheral

The two protocols have completely different purposes.

Protocol	Purpose
APB	Configure and control the SPI controller
SPI	Transfer serial data to/from an external peripheral

⸻

3. Overall Architecture

The SPI core consists of four major blocks.

                         APB MASTER
                             |
             +---------------+----------------+
             |                                |
             v                                |
      +--------------+                        |
      | APB SLAVE    |                        |
      | INTERFACE    |                        |
      +------+-------+                        |
             |                                |
       Configuration                         |
       and Data                              |
             |                                |
     +-------+-------------------------------+
     |
     +------------------+
     |                  |
     v                  v
+-----------+     +-------------+
| Baud Rate |     | Slave Select|
| Generator |     | Generator   |
+-----+-----+     +------+------+
      |                  |
     SCLK                SS
      |                  |
      +--------+---------+
               |
               v
        +-------------+
        | SPI SHIFTER |
        +------+------+
               |
          +----+----+
          |    |    |
         MOSI MISO SCLK
               |
               |
              SS

Each block has a specific responsibility.

APB Slave Interface

Handles communication with the APB master and provides access to SPI registers.

Baud Rate Generator

Generates the SPI serial clock SCLK from the APB clock PCLK.

Slave Select Generator

Generates the active-low SS signal and controls the duration of an SPI transaction.

SPI Shifter

Converts parallel data into serial data for MOSI and converts serial MISO data back into parallel data.

⸻

4. Understanding the Master and Slave Roles

This is one of the most important concepts in the project.

There are actually two different master/slave relationships.

4.1 APB Master and APB Slave

On the APB side:

APB MASTER
   |
   | PADDR
   | PWRITE
   | PWDATA
   | PSEL
   | PENABLE
   v
APB SLAVE INTERFACE
   |
   | PRDATA
   | PREADY
   | PSLVERR
   v
APB MASTER

The processor or system bus controller is the APB Master.

The SPI controller’s APB interface is the APB Slave.

The SPI controller does not initiate APB transactions.

It waits for the APB master to access its registers.

⸻

4.2 SPI Master and SPI Slave

On the SPI side, the SPI controller can operate as the SPI Master.

In master mode, it generates:

* SCLK
* SS

and transmits:

* MOSI

while receiving:

* MISO

Conceptually:

             SPI MASTER
          +-------------+
          | SPI CORE    |
          +-------------+
             | | | |
             | | | +------ SS
             | | +-------- SCLK
             | +---------- MOSI
             +------------ MISO
                  |
                  v
             SPI SLAVE
          +-------------+
          | Peripheral  |
          +-------------+

Therefore:

APB side:
Processor = APB Master
SPI Core  = APB Slave
SPI side:
SPI Core       = SPI Master
External device = SPI Slave

This distinction is essential.

⸻

5. APB Side

The SPI controller is accessed through APB.

The main APB signals are:

Signal	Direction	Purpose
PCLK	Input	APB clock
PRESETn	Input	Active-low reset
PSEL	Input	Selects the SPI APB slave
PENABLE	Input	Indicates APB enable phase
PWRITE	Input	Indicates read/write operation
PADDR	Input	Register address
PWDATA	Input	Write data
PRDATA	Output	Read data
PREADY	Output	Indicates transfer completion
PSLVERR	Output	Indicates APB error

The APB transaction has three conceptual phases:

IDLE
  |
  v
SETUP
  |
  v
ENABLE
  |
  v
IDLE

The SPI APB interface implements these states internally.

⸻

6. SPI Side

The SPI interface uses four primary signals:

SS
SCLK
MOSI
MISO

SCLK

Serial clock generated by the SPI master.

MOSI

Master Out Slave In.

Data travels:

SPI Master ---> SPI Slave

MISO

Master In Slave Out.

Data travels:

SPI Slave ---> SPI Master

SS

Slave Select.

It is active low in this implementation.

SS = 0  -> Slave selected
SS = 1  -> Slave deselected

⸻

7. Complete Data Flow

Suppose software wants to transmit:

1010_1100

The sequence is:

APB MASTER
    |
    | Write 10101100
    | to SPI Data Register
    v
APB SLAVE INTERFACE
    |
    | SPI_DR
    v
Transmit Control
    |
    | send_data
    v
Slave Select Generator
    |
    | SS = 0
    v
Baud Rate Generator
    |
    | SCLK
    v
SPI Shifter
    |
    | MOSI
    v
SPI SLAVE

At the same time, the SPI slave can return data through MISO.

SPI SLAVE
    |
    | MISO
    v
SPI SHIFTER
    |
    | received byte
    v
SPI DATA REGISTER
    |
    | APB read
    v
APB MASTER

This is why SPI is called a full-duplex interface.

Transmission and reception happen during the same clocked transaction.

⸻

8. SPI Controller Register Map

The register map defined for the SPI core is:

Address	Register	Access
0	SPI Control Register 1	RW
1	SPI Control Register 2	RW
2	SPI Baud Rate Register	RW
3	SPI Status Register	RO
5	SPI Data Register	RW

The register address is represented using a 3-bit APB address.

PADDR[2:0]

Example:

PADDR = 3'b000 -> SPI_CR1
PADDR = 3'b001 -> SPI_CR2
PADDR = 3'b010 -> SPI_BR
PADDR = 3'b011 -> SPI_SR
PADDR = 3'b101 -> SPI_DR

⸻

9. SPI Control Register 1

Register:

SPI_CR1

Bit arrangement:

+------+-----+-------+------+-----+-----+------+------+
| SPIE | SPE | SPTIE | MSTR | CPOL| CPHA| SSOE |LSBFE |
+------+-----+-------+------+-----+-----+------+------+
   7     6      5       4     3     2     1      0

SPIE

SPI interrupt enable.

Enables interrupt generation associated with transfer completion and mode fault conditions.

SPE

SPI system enable.

Enables the SPI system.

SPTIE

SPI transmit interrupt enable.

Enables interrupt generation associated with the transmit buffer status.

MSTR

Master mode enable.

MSTR = 1

configures the SPI core as a master.

CPOL

Clock polarity.

Determines the idle polarity of SCLK.

CPOL = 0 -> SCLK idle low
CPOL = 1 -> SCLK idle high

CPHA

Clock phase.

Determines which clock edge is used for data timing.

SSOE

Slave-select output enable.

The manual specifies that the SS output feature is enabled in master mode.

LSBFE

LSB-first enable.

LSBFE = 0 -> MSB first
LSBFE = 1 -> LSB first

Reset value:

8'b0000_0100

⸻

10. SPI Control Register 2

Register:

SPI_CR2

Bit arrangement:

+---+---+---+--------+---------+---+---------+------+
| 0 | 0 | 0 | MODFEN | BIDIROE | 0 | SPISWAI | SPC0 |
+---+---+---+--------+---------+---+---------+------+

MODFEN

Mode fault detection enable.

Allows mode fault detection.

BIDIROE

Bidirectional mode output enable.

Controls output enable during bidirectional operation.

SPISWAI

SPI wait-in-stop control.

Used for power conservation while operating in wait mode.

SPC0

Serial pin control bit.

Reset value:

8'b0000_0000

⸻

11. SPI Baud Rate Register

Register:

SPI_BR

Bit arrangement:

+---+------+------+------+---+------+------+------+
| 0 |SPPR2 |SPPR1 |SPPR0 | 0 | SPR2 | SPR1 | SPR0 |
+---+------+------+------+---+------+------+------+

Two fields control the baud-rate divisor.

SPPR

SPPR[2:0]

SPI Baud Rate Preselection Bits.

SPR

SPR[2:0]

SPI Baud Rate Selection Bits.

The divisor is calculated as:

Baud Rate Divisor = (SPPR + 1) × 2^(SPR + 1)

The SPI clock frequency is:

SPI Baud Rate = PCLK / Baud Rate Divisor

For example, if:

PCLK = 100 MHz
Baud Rate Divisor = 20

then:

SCLK = 100 MHz / 20
     = 5 MHz

⸻

12. SPI Status Register

Register:

SPI_SR

Bit arrangement:

+------+---+-------+------+---+---+---+---+
| SPIF | 0 | SPTEF | MODF | 0 | 0 | 0 | 0 |
+------+---+-------+------+---+---+---+---+

SPIF

SPI transfer complete flag.

Indicates that received data has been transferred into the SPI data register.

SPTEF

SPI transmit buffer empty flag.

Indicates that the transmit data register is empty.

MODF

Mode fault flag.

Indicates a mode fault condition when the appropriate master-mode and mode-fault conditions occur.

Reset value:

8'b0010_0000

⸻

13. SPI Data Register

Register:

SPI_DR

Width:

8 bits
+---+---+---+---+---+---+---+---+
| 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
+---+---+---+---+---+---+---+---+

This register has two purposes:

Write

APB master writes transmit data.

APB Master
    |
    | PWDATA
    v
 SPI_DR

Read

Received SPI data is stored in the register and can then be read through APB.

SPI Slave
    |
   MISO
    |
    v
Shifter
    |
    v
 SPI_DR
    |
    | PRDATA
    v
APB Master

⸻

14. SPI Operating Modes

The controller uses three internal SPI operating states:

SPI_RUN
SPI_WAIT
SPI_STOP

SPI Run Mode

Normal SPI operation.

The SPI core can generate SCLK and perform transfers when the required control signals are active.

SPI Wait Mode

SPI operation depends on the SPISWAI configuration.

The purpose is related to power conservation while the system is in wait mode.

SPI Stop Mode

SPI operation is stopped.

The baud-rate generator does not continue normal SCLK generation.

⸻

15. Block 1: APB Slave Interface

The APB Slave Interface is the central control block.

Its job is to:

1. Receive APB transactions.
2. Decode APB addresses.
3. Perform register writes.
4. Perform register reads.
5. Generate APB response signals.
6. Decode SPI configuration fields.
7. Generate SPI operating modes.
8. Generate status flags.
9. Generate interrupt requests.
10. Connect the SPI registers with the other three blocks.

⸻

15.1 APB FSM

The interface uses three states:

IDLE
SETUP
ENABLE

IDLE

No APB transaction is currently being processed.

SETUP

The APB master selects the peripheral and provides the address/control information.

ENABLE

The actual APB transfer takes place.

Conceptually:

       PSEL
        |
        v
      SETUP
        |
     PENABLE
        |
        v
      ENABLE
        |
        v
       IDLE

⸻

15.2 Write Enable

The write enable is asserted when:

PWRITE = 1

and the APB FSM is in:

ENABLE

Conceptually:

wr_enb = PWRITE && ENABLE

This tells the register logic that a valid APB write is occurring.

⸻

15.3 Read Enable

Similarly:

rd_enb = !PWRITE && ENABLE

This indicates a valid APB read.

⸻

15.4 Register Write Operation

When:

wr_enb = 1

the address determines which register receives PWDATA.

For example:

PADDR = 000 -> SPI_CR1
PADDR = 001 -> SPI_CR2
PADDR = 010 -> SPI_BR
PADDR = 101 -> SPI_DR

⸻

15.5 Register Read Operation

During a read transaction:

rd_enb = 1

the address is decoded.

The corresponding register is placed on:

PRDATA

Conceptually:

PADDR
  |
  v
+----------------+
| Address Decode |
+----------------+
  |
  +---- SPI_CR1
  |
  +---- SPI_CR2
  |
  +---- SPI_BR
  |
  +---- SPI_SR
  |
  +---- SPI_DR

⸻

15.6 Configuration Field Decoding

The APB interface extracts individual fields from the control registers.

For example:

SPI_CR1[4] -> MSTR
SPI_CR1[3] -> CPOL
SPI_CR1[2] -> CPHA
SPI_CR1[0] -> LSBFE

These signals are passed to the appropriate SPI blocks.

Similarly:

SPI_CR2[1] -> SPISWAI
SPI_CR2[4] -> MODFEN

and:

SPI_BR[6:4] -> SPPR
SPI_BR[2:0] -> SPR

⸻

16. Block 2: Baud Rate Generator

The Baud Rate Generator converts the APB clock into the SPI serial clock.

Input:

PCLK

Output:

SCLK

Conceptually:

             PCLK
              |
              v
       +--------------+
       | Baud Rate    |
       | Generator    |
       +------+-------+
              |
             SCLK

⸻

16.1 Baud Rate Calculation

The divisor is:

Baud Rate Divisor = (SPPR + 1) × 2^(SPR + 1)

Then:

SCLK frequency = PCLK / Baud Rate Divisor

⸻

16.2 Internal Counter

The generator uses a counter to divide the APB clock.

The basic concept is:

PCLK PCLK PCLK PCLK PCLK ...
 |    |    |    |    |
 +----+----+----+----+
          counter
             |
       terminal count
             |
             v
        toggle SCLK

The manual describes the SCLK generation using a half-period count.

The counter reaches:

BaudRateDivisor / 2 - 1

and the SCLK toggles.

⸻

16.3 CPOL and Initial SCLK

The initial SCLK state is determined by CPOL.

CPOL = 0 -> initial SCLK = 0
CPOL = 1 -> initial SCLK = 1

When the SPI transaction is inactive, SCLK returns to its initial polarity.

⸻

16.4 When Does SCLK Run?

SCLK generation is enabled when the appropriate conditions are satisfied:

SS active
AND
SPI operating mode is valid
AND
SPISWAI does not prevent operation

Otherwise:

SCLK -> idle polarity
counter -> reset

⸻

16.5 MISO Sampling Flags

The baud generator also produces timing flags that tell the shifter when MISO should be sampled.

The sampling edge depends on:

CPOL
CPHA

The combinations are:

CPOL	CPHA	Sampling Edge
0	0	Rising
0	1	Falling
1	0	Falling
1	1	Rising

The baud generator converts these edge requirements into internal timing pulses.

⸻

16.6 MOSI Timing Flags

The baud generator also produces timing flags for MOSI transmission.

The shifter uses these signals to determine when the next MOSI bit should be presented.

This separates the responsibilities:

Baud Generator
      |
      | timing
      v
Shifter
      |
      | data
      v
     MOSI

The baud generator does not decide which data bit should be transmitted.

It only generates the timing.

⸻

17. Block 3: SPI Slave Select Generator

The Slave Select Generator controls:

SS

It also generates:

receive_data
tip

Conceptually:

send_data
    |
    v
+----------------------+
| Slave Select         |
| Generator            |
+----------------------+
    |       |       |
    v       v       v
   SS    receive   tip
          _data

⸻

17.1 Starting a Transaction

When new transmit data is available:

send_data = 1

and the SPI controller is operating as a master, the module asserts:

SS = 0

This selects the external SPI slave.

⸻

17.2 Transaction Timing

The SS signal remains low for the required transfer period.

The generator uses a counter based on the baud-rate divisor.

The manual describes a target timing value derived as:

target = BaudRateDivisor × 16

The counter runs until the target duration has been reached.

⸻

17.3 Completing the Transaction

After the required number of clock periods:

SS -> 1

The SPI slave is deselected.

The transfer is therefore:

SS = 1
       |
       | start
       v
SS = 0
       |
       | SPI clocks + data transfer
       |
       v
SS = 1

⸻

17.4 Transfer-In-Progress

The tip signal indicates whether an SPI transfer is active.

The manual defines it based on the SS state:

tip = ~SS

Therefore:

SS  = 1 -> tip = 0
SS  = 0 -> tip = 1

So:

tip = 1

means:

SPI transfer is currently in progress.

⸻

17.5 Receive Data Signal

After the appropriate transfer duration, the module generates:

receive_data

This informs the rest of the SPI core that received data can be captured into the data register.

⸻

18. Block 4: SPI Shifter

The shifter is responsible for the actual serial data movement.

It performs two jobs:

Transmit

Parallel data:

8'b1010_1100

becomes serial data:

1 -> 0 -> 1 -> 0 -> 1 -> 1 -> 0 -> 0

or the reverse order when LSB-first is enabled.

Receive

Serial data:

1 -> 1 -> 0 -> 0 -> 1 -> 0 -> 1 -> 0

is collected into:

8'b1100_1010

⸻

18.1 Internal Registers

The shifter contains internal storage such as:

shift_register
temp_reg

and counters for transmission and reception.

shift_register

Stores the outgoing 8-bit data.

temp_reg

Collects the incoming MISO data.

⸻

19. CPOL and CPHA

SPI supports four standard clock modes.

They are determined by:

CPOL
CPHA

SPI Mode	CPOL	CPHA
Mode 0	0	0
Mode 1	0	1
Mode 2	1	0
Mode 3	1	1

⸻

Mode 0

CPOL = 0
CPHA = 0

SCLK is idle low.

The first active edge is rising.

⸻

Mode 1

CPOL = 0
CPHA = 1

SCLK is idle low.

The first sampling relationship changes compared with Mode 0.

⸻

Mode 2

CPOL = 1
CPHA = 0

SCLK is idle high.

The active edge relationship is inverted relative to Mode 0.

⸻

Mode 3

CPOL = 1
CPHA = 1

SCLK is idle high.

The timing relationship changes according to CPHA.

The shifter and baud generator use these two bits to determine when data should be transmitted and sampled.

⸻

20. MSB-First and LSB-First

The LSBFE bit determines the bit order.

MSB First

If:

LSBFE = 0

data is transferred:

bit 7
bit 6
bit 5
bit 4
bit 3
bit 2
bit 1
bit 0

For:

1010_1100

the transmission order is:

1 0 1 0 1 1 0 0
^
MSB

⸻

LSB First

If:

LSBFE = 1

data is transferred:

bit 0
bit 1
bit 2
bit 3
bit 4
bit 5
bit 6
bit 7

For:

1010_1100

the transmission order becomes:

0 0 1 1 0 1 0 1

The shifter contains separate counting/timing logic to support these two directions.

⸻

21. SPI Transaction

A complete SPI transaction can be understood in five stages.

Stage 1: Load Data

The APB master writes data:

PWDATA = 8'b1010_1100

to:

SPI_DR

⸻

Stage 2: Select Slave

The Slave Select Generator detects new transmit data.

SS = 0

The external SPI slave is selected.

⸻

Stage 3: Generate Clock

The Baud Rate Generator starts generating:

SCLK

according to:

SPPR
SPR
CPOL
CPHA

⸻

Stage 4: Shift Data

The SPI Shifter:

MOSI -> transmit data
MISO -> receive data

Data is transferred according to the configured SPI mode.

⸻

Stage 5: End Transaction

After the required transfer duration:

SS = 1

The transaction ends.

The received data is placed into the SPI data register.

⸻

22. Read Transaction

An APB read retrieves data from the SPI controller.

The basic sequence is:

APB MASTER
    |
    | PSEL = 1
    | PWRITE = 0
    | PADDR = register address
    v
APB SLAVE
    |
    | decode address
    v
Register
    |
    | PRDATA
    v
APB MASTER

For example:

PADDR = 3'b011

selects the SPI status register.

The status register contents are placed on:

PRDATA

⸻

23. Write Transaction

For a write:

PWRITE = 1

The APB master supplies:

PADDR
PWDATA

Example:

PADDR  = 3'b101
PWDATA = 8'hA5

This writes:

SPI_DR = 8'hA5

The write then initiates the SPI transmission process.

⸻

24. Transfer Completion

The SPI data path uses internal control signals to determine when a transaction starts and ends.

Conceptually:

APB Write
    |
    v
SPI_DR
    |
    v
send_data
    |
    v
SS = 0
    |
    v
SCLK generation
    |
    v
MOSI/MISO shifting
    |
    v
receive_data
    |
    v
SPI_DR updated
    |
    v
SS = 1

The status logic then reflects the transfer state.

⸻

25. Interrupt Generation

The SPI core supports interrupt generation through:

SPIE
SPTIE

and status conditions:

SPIF
SPTEF
MODF

The interrupt logic follows these combinations.

Neither interrupt enable active

SPIE  = 0
SPTIE = 0
interrupt = 0

SPIE active

SPIE  = 1
SPTIE = 0

Interrupt is generated when:

SPIF || MODF

is active.

SPTIE active

SPIE  = 0
SPTIE = 1

Interrupt follows:

SPTEF

Both active

SPIE  = 1
SPTIE = 1

Interrupt is generated when any relevant status condition is active:

SPIF || MODF || SPTEF

Conceptually:

             +------ SPIF
             |
             +------ MODF ----+
             |                |
             +------ SPTEF ---+----> Interrupt
                              |
                         Enable Logic

⸻

26. Mode Fault Detection

Mode fault detection is associated with master operation.

The relevant configuration includes:

MSTR
MODFEN
SS
SSOE

A mode fault can occur when the SPI is configured as a master, mode-fault detection is enabled, and the slave-select condition indicates a fault.

The resulting status condition is:

MODF = 1

The interrupt logic can then use MODF when SPIE is enabled.

⸻

27. Reset Behavior

The design uses an active-low reset:

PRESETn

Reset is asserted when:

PRESETn = 0

The major registers and internal state machines are returned to their reset states.

Typical reset behavior includes:

SPI_CR1 -> 8'b0000_0100
SPI_CR2 -> 8'b0000_0000
SPI_BR  -> 8'b0000_0000
SPI_SR  -> 8'b0010_0000
SPI_DR  -> 8'b0000_0000

The timing blocks also return to their inactive state.

For example:

SS   -> 1
SCLK -> CPOL-defined idle state

The shifter registers are cleared.

⸻

28. RTL Block Interaction

The most important part of understanding the implementation is understanding how the blocks communicate.

A simplified signal-level relationship is:

                         APB MASTER
                             |
                             |
                 PADDR/PWRITE/PWDATA
                             |
                             v
                 +----------------------+
                 | APB SLAVE INTERFACE  |
                 +----------------------+
                    |       |       |
             config |       |       | data
                    |       |       |
          +---------+       |       +---------+
          |                 |                 |
          v                 v                 v
       CPOL/CPHA         Baud Rate          SPI_DR
       MSTR/LSBFE        settings             |
       SPPR/SPR                               |
          |                                   |
          v                                   v
 +------------------+                +----------------+
 | Baud Generator  |                | Transfer Ctrl |
 +--------+---------+                +-------+--------+
          |                                  |
         SCLK                                |
          |                                  |
          |                                 SS
          |                                  |
          +----------------+-----------------+
                           |
                           v
                   +---------------+
                   | SPI SHIFTER   |
                   +-------+-------+
                       |       |
                      MOSI    MISO
                       |       |
                       v       |
                   SPI SLAVE --+

The division of responsibility is important:

APB Interface
     |
     +-- Configuration
     +-- Register access
     +-- Status
     +-- Interrupts
Baud Generator
     |
     +-- SCLK timing
     +-- MISO timing flags
     +-- MOSI timing flags
Slave Select Generator
     |
     +-- SS
     +-- Transfer timing
     +-- receive_data
     +-- tip
Shifter
     |
     +-- MOSI data
     +-- MISO data
     +-- Bit ordering
     +-- CPOL/CPHA-dependent shifting

⸻

29. Directory Structure

The project follows the general structure used for the SPI design exercises:

spi_design_lab/
│
├── rtl/
│   ├── apb_slave.v
│   ├── baud_generator.v
│   ├── spi_slave_select.v
│   ├── spi_shifter.v
│   └── spi_top.v
│
├── tb/
│   ├── apb_slave_tb.v
│   ├── baud_generator_tb.v
│   ├── spi_slave_select_tb.v
│   ├── spi_shifter_tb.v
│   └── spi_top_tb.v
│
├── sim/
│   └── simulation files
│
├── synth/
│   └── synthesis results
│
├── lint/
│   └── lint reports
│
└── README.md

The exact filenames can be changed according to the final RTL implementation.

⸻

30. Verification

Each major RTL block should be verified independently before integrating the complete SPI controller.

The verification strategy is:

             Individual Verification
                     |
       +-------------+-------------+
       |             |             |
       v             v             v
     Baud          Slave          APB
    Generator     Select         Interface
       |             |             |
       +-------------+-------------+
                     |
                     v
                  Shifter
                     |
                     v
              Full SPI Core
                     |
                     v
              System Verification

⸻

30.1 Baud Rate Generator Testbench

Verify:

* reset behavior
* divisor calculation
* SCLK generation
* CPOL behavior
* run mode
* wait mode
* stop mode
* SCLK frequency
* MOSI timing flags
* MISO timing flags

⸻

30.2 Slave Select Generator Testbench

Verify:

* reset
* SS idle state
* master mode
* send_data detection
* SS assertion
* SS duration
* receive_data generation
* transfer-in-progress indication

⸻

30.3 APB Slave Interface Testbench

Verify:

* APB reset
* APB state transitions
* register writes
* register reads
* address decoding
* control register configuration
* baud-rate register
* data register
* status register
* interrupt request
* mode fault

⸻

30.4 Shifter Testbench

Verify:

* reset
* transmit data loading
* MOSI generation
* MISO sampling
* MSB-first
* LSB-first
* CPOL = 0
* CPOL = 1
* CPHA = 0
* CPHA = 1
* complete 8-bit transfers

⸻

31. Simulation

A typical simulation flow is:

Compile RTL
     |
     v
Compile Testbench
     |
     v
Start Simulation
     |
     v
Apply Reset
     |
     v
Configure SPI
     |
     v
Write SPI Data Register
     |
     v
Observe SS
     |
     v
Observe SCLK
     |
     v
Observe MOSI/MISO
     |
     v
Check Received Data
     |
     v
Check Status

Important waveforms to observe:

PCLK
PRESETn
PSEL
PENABLE
PWRITE
PADDR
PWDATA
PRDATA
PREADY
PSLVERR
SS
SCLK
MOSI
MISO
send_data
receive_data
tip
SPIF
SPTEF
MODF
spi_interrupt_request

⸻

32. Lint

RTL linting is used to identify coding and structural problems before synthesis.

Typical issues to check include:

* incomplete assignments
* inferred latches
* multiple drivers
* unused signals
* width mismatches
* incorrect sensitivity lists
* unreachable states
* combinational loops
* sequential/combinational mixing
* reset inconsistencies
* signed/unsigned problems

The intended RTL should be synthesizable and lint-clean.

⸻

33. Synthesis

After simulation and linting, the RTL can be synthesized.

The flow is:

Verilog RTL
    |
    v
Lint
    |
    v
Simulation
    |
    v
Synthesis
    |
    v
Gate-Level Netlist

The synthesized design can then be inspected for:

* cell usage
* area
* timing
* logic structure
* critical paths

Gate-level simulation can also be performed if required.

⸻

34. Key Design Equations

Baud Rate Divisor

Baud Rate Divisor = (SPPR + 1) × 2^(SPR + 1)

SPI Clock Frequency

SCLK = PCLK / Baud Rate Divisor

SCLK Half Period

The clock generator uses a half-period counter based on:

Baud Rate Divisor / 2

The implementation therefore toggles SCLK after the corresponding half-period count.

Slave Select Target

The slave-select timing logic uses:

target = BaudRateDivisor × 16

as described in the design manual.

⸻

35. Important Signals

APB Signals

PCLK
PRESETn
PSEL
PENABLE
PWRITE
PADDR
PWDATA
PRDATA
PREADY
PSLVERR

SPI Signals

SCLK
MOSI
MISO
SS

Configuration Signals

MSTR
SPE
SPIE
SPTIE
CPOL
CPHA
SSOE
LSBFE
MODFEN
BIDIROE
SPISWAI
SPC0
SPPR
SPR

Internal Control Signals

send_data
receive_data
tip
spif
sptef
modf
spi_interrupt_request

⸻

36. Design Limitations and Scope

This project follows the architecture and behavior specified by the Maven Silicon SPI design exercise.

The main implementation scope is:

* APB3 slave interface
* SPI controller operation
* Master-mode SPI transfer
* 8-bit data transfer
* CPOL/CPHA support
* MSB-first and LSB-first operation
* Programmable baud rate
* Slave-select generation
* SPI status reporting
* Interrupt generation
* Mode fault detection
* Synthesizable Verilog RTL

The exact supported behavior should be considered according to the implemented RTL and the provided design specification.

⸻

37. Learning Objectives

This project is designed to provide practical understanding of several RTL and VLSI concepts.

Protocol Understanding

* AMBA APB3
* SPI
* APB master/slave relationship
* SPI master/slave relationship
* SPI clock modes
* serial data transfer

RTL Design

* FSM design
* sequential logic
* combinational logic
* counters
* register maps
* address decoding
* clock generation
* shift registers
* reset handling

Verification

* Verilog testbench development
* waveform analysis
* functional checking
* protocol-level verification
* block-level verification

VLSI Flow

* RTL coding
* lint
* simulation
* synthesis
* gate-level netlist generation

⸻

38. References

The primary design reference for this implementation is the Maven Silicon APB-Based SPI Protocol / SPI Manual, which defines the project architecture, register map, SPI control fields, baud-rate generation, slave-select generation, APB interface behavior, and shifter implementation requirements.

Primary reference:

Maven Silicon, Serial Peripheral Interface / APB Based SPI Protocol, 2025.

⸻

Project Summary

At its core, this project implements the following chain:

                  SOFTWARE / PROCESSOR
                          |
                          | APB
                          v
                +---------------------+
                |   APB SLAVE         |
                |   INTERFACE         |
                |                     |
                | Control Registers   |
                | Status Registers    |
                | Data Register       |
                +----------+----------+
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
        Configuration   Baud Rate    Transfer Data
             |          Generator         |
             |             |              |
             |            SCLK            |
             |             |              |
             +-------------+--------------+
                           |
                           v
                  +----------------+
                  | SPI SHIFTER    |
                  +-------+--------+
                          |
                    +-----+-----+
                    |     |     |
                   SS    MOSI  MISO
                    |     |     |
                    +-----+-----+
                          |
                          v
                    SPI PERIPHERAL

The key idea is simple:

APB controls the SPI controller.
The Baud Rate Generator creates the SPI clock.
The Slave Select Generator controls the transaction.
The Shifter moves the actual SPI data.

That separation of responsibilities is the foundation of this project.
