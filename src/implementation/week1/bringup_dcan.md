# Overview

This phase of the project focused on bringing up the AM335x D-CAN controller on the BeagleBone Black running RTEMS.

The primary objective was not yet to transmit CAN frames, but rather to establish and verify the complete software-to-hardware interaction path required for future driver development.

Starting from a minimal RTEMS application, I progressively verified:

```text
RTEMS Application
    ↓
ARM CPU Execution
    ↓
Memory-Mapped Register Access
    ↓
DCAN Controller Configuration
    ↓
Interface Registers (IF1)
    ↓
Message RAM
    ↓
Message Object Configuration
```

During this process, I gained practical experience with several fundamental embedded systems concepts, including:

```text
Memory-mapped I/O
Address spaces and register offsets
Clock gating
Synchronous digital logic
D Flip-Flops
CAN bit timing
Interface registers
Message RAM architecture
RTEMS application deployment
```

One of the most important realizations was developing an intuitive understanding of how software actually reaches and executes on embedded hardware.

Before this project, I understood that RTEMS applications could be compiled into executable images, but I did not have a concrete mental picture of how software traveled from source code to a running application on the BeagleBone Black.

Through repeatedly building, deploying, and testing the application, I established the following mental model:

```text
Source Code
    ↓
RTEMS Build System
    ↓
rtems-app.img
    ↓
TFTP Transfer
    ↓
U-Boot Bootloader
    ↓
BeagleBone Black
    ↓
ARM Cortex-A8 Processor
    ↓
Memory-Mapped Registers
    ↓
Hardware Peripheral (DCAN)
```

This understanding significantly reduced the abstraction barrier between software and hardware and provided a strong foundation for future RTEMS driver development.

The work completed in this phase successfully verified:

```text
✓ RTEMS application execution on BBB
✓ DCAN base address access
✓ DCAN clock enable
✓ Controller register access
✓ Initialization/configuration mode
✓ Bit Timing Register (BTR) access
✓ TX Message Object configuration
✓ IF1 → Message RAM transfer
✓ Message RAM → IF1 readback transfer
✓ Message Object verification
```

The next stage of development will focus on actual CAN communication by requesting frame transmission, connecting external CAN hardware, and integrating the implementation into the RTEMS CAN driver framework.

---

# A Key Realization

One of my biggest realizations was that RTEMS development is not just reading source code.

At the beginning of this project, I understood that RTEMS applications could be compiled into executable images, but I did not have a concrete mental picture of how software actually reached the BeagleBone Black and interacted with hardware.

Through this workflow:

```text
Source Code
    ↓
RTEMS Build
    ↓
RTEMS Application Image
    ↓
TFTP Transfer
    ↓
U-Boot
    ↓
BeagleBone Black
    ↓
ARM CPU Execution
    ↓
Hardware Registers
```

I finally developed an intuitive understanding that:

> The code I write on my Mac can actually run on the ARM processor inside the BeagleBone Black and directly control hardware peripherals.

This was an important mental breakthrough.

---

# Understanding Memory-Mapped Hardware

The first step was learning that hardware peripherals appear as memory addresses.

For example:

```text
DCAN0 = 0x481CC000
DCAN1 = 0x481D0000
```

These are not ordinary variables.

Instead:

```text
Specific memory addresses
↔
Specific hardware registers
```

Reading:

```c
REG32(SOC_DCAN_1_REGS + DCAN_CTL)
```

means:

```text
Read the DCAN Control Register directly from hardware.
```

This was my first successful hardware register access on the BBB.

---

# Understanding Addressing

Another important concept was understanding memory addresses.

Initially I confused:

```text
Address increments
```

with:

```text
Bit-width increments
```

I learned that:

```text
Addresses are byte-addressable.
```

For example:

```text
0x1000
0x1001
0x1002
0x1003
```

represent:

```text
4 consecutive bytes
```

not:

```text
4 bits
```

A 32-bit register occupies:

```text
4 bytes
```

therefore:

```text
Register N
→ address + 4
→ Register N+1
```

This explained why many DCAN registers appear at offsets:

```text
0x0
0x4
0x8
0xC
```

inside the TRM.

A related realization was that:

```text
Hexadecimal digits describe numbers.
Addresses describe locations.
```

Address changes are not related to the number of bits displayed in hexadecimal notation.

---

# Understanding Clock Enable

Another important concept was understanding why peripherals require clocks.

Initially I thought:

```text
Registers always exist.
```

which is true physically.

However:

```text
The internal logic cannot operate without a clock.
```

The clock acts like a heartbeat for digital logic.

Most internal state updates occur on clock edges.

Without a clock:

```text
State machines stop
Counters stop
Register updates stop
Peripheral logic stops
```

Therefore the first requirement before accessing DCAN was:

```text
Enable DCAN1 clock
```

through:

```text
CM_PER_DCAN1_CLKCTRL
```

After enabling the clock, register access became reliable.

---

# Understanding Synchronous Digital Logic

While studying clock behavior, I also learned a fundamental digital design principle:

```text
Most digital systems are synchronous.
```

A D Flip-Flop behaves as:

```text
Clock Rising Edge
        ↓
Sample Input D
        ↓
Update Output Q
```

This means the entire hardware system updates state in a coordinated way.

This concept helped me better understand:

```text
Registers
State Machines
Microcontrollers
Peripherals
```

at the hardware level.

---

# Reading Controller Registers

After enabling the clock, I successfully accessed:

```text
DCAN_CTL
DCAN_ES
```

and verified:

```text
RTEMS
→ Memory Mapped I/O
→ DCAN Hardware
```

was working correctly.

Observed values:

```text
DCAN1 CTL = 0x00001401
DCAN1 ES  = 0x00000207
```

This was the first successful interaction with live DCAN hardware registers.

---

# Understanding CAN Bit Timing

One concept I learned was that a CAN bit is not simply:

```text
0
or
1
```

Instead, each CAN bit is divided into multiple:

```text
Time Quanta (TQ)
```

A CAN bit consists of:

```text
SYNC_SEG
PROP_SEG
PHASE_SEG1
PHASE_SEG2
```

These segments allow CAN controllers on different nodes to synchronize themselves and tolerate clock differences.

This timing is configured through:

```text
BTR
(Bit Timing Register)
```

which controls:

```text
BRP
SJW
TSEG1
TSEG2
```

and ultimately determines the CAN baud rate.

This was my first exposure to the fact that communication timing is configured at the hardware level rather than being handled automatically by software.

---

# Controller Initialization

Following the AM335x TRM procedure:

1. Set INIT
2. Set CCE
3. Configure BTR
4. Clear INIT and CCE

I successfully moved the controller into and out of configuration mode.

Observed values:

```text
Before:
CTL = 0x00001401

After entering config mode:
CTL = 0x00001441

After leaving config mode:
CTL = 0x00001400
```

This verified:

```text
Controller configuration path works.
```

---

# Understanding Message Objects

The most important new concept was learning the Message RAM architecture.

Unlike normal memory:

```text
CPU cannot directly access Message RAM.
```

Instead, DCAN uses:

```text
IF1
IF2
```

Interface Registers.

The data flow is:

```text
CPU
    ↓
IF1
    ↓
Message RAM
    ↓
Message Object
```

This was a completely new hardware architecture for me.

---

# Understanding Acceptance Masks

I also learned the purpose of message masks.

A mask determines:

```text
Which CAN identifiers should match.
```

For example:

```text
Mask = all ones
```

means:

```text
Exact identifier matching.
```

While:

```text
Mask = some zeros
```

allows a range of identifiers to match.

For the initial TX object experiment:

```text
Mask = 0
```

because transmit objects do not require acceptance filtering.

---

# Configuring a TX Message Object

I configured:

```text
Message Object 1
```

with:

```text
ID  = 0x123
DLC = 8

Data:
11 22 33 44
55 66 77 88
```

using:

```text
IF1MSK
IF1ARB
IF1MCTL
IF1DATA
IF1DATB
IF1CMD
```

The IF1 command register then transferred these values into Message Object 1.

This established the path:

```text
CPU
→ IF1
→ Message RAM
→ Message Object 1
```

---

# Reading Back Message Object Data

To verify the configuration, I performed a readback operation.

The path was:

```text
Message Object 1
    ↓
IF1
    ↓
CPU
```

The readback values matched the original values exactly:

```text
IF1ARB  = 0xA48C0000
IF1MCTL = 0x00000088
IF1DATA = 0x44332211
IF1DATB = 0x88776655

Decoded ID  = 0x123
Decoded DLC = 8
```

This confirmed:

```text
CPU ↔ IF1 ↔ Message RAM
```

communication was functioning correctly.

---

# Current Status

The following functionality has been verified:

```text
✓ RTEMS boot on BBB
✓ TFTP deployment workflow
✓ DCAN base address access
✓ Clock enable
✓ Controller register access
✓ INIT / CCE configuration
✓ BTR access
✓ Message Object configuration
✓ IF1 write transfer
✓ IF1 readback transfer
✓ Message RAM verification
```

---

# Lessons Learned

Through this work I gained a much clearer understanding of:

```text
Memory-mapped I/O
Hardware registers
Clock gating
Synchronous digital logic
D Flip-Flops
CAN bit timing
Message RAM architecture
Interface registers
Embedded software deployment
```

Most importantly, I now have a concrete understanding that:

```text
RTEMS application
    ↓
ARM processor
    ↓
Memory mapped registers
    ↓
Real hardware behavior
```

This bridge between software and hardware was one of the most valuable lessons of the project so far.

---

# Next Steps

The next milestone is:

```text
Set TXRQST
    ↓
Request frame transmission
    ↓
Connect CAN transceiver
    ↓
Receive frame using Linux SocketCAN
    ↓
Implement RTEMS DCAN driver
```

This will move the project from controller configuration into actual CAN communication.