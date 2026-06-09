# Functional Driver Bring-up: From Register Access to CAN Frame Communication on AM335x D-CAN

## Overview

Driver bring-up is often an iterative process.

Instead of implementing a complete driver immediately, I gradually verified every layer of the hardware and software stack until a complete CAN communication path was established.

This post summarizes the bring-up methodology used for the AM335x D-CAN controller on BeagleBone Black.

---

# Step 1. Verify Register Access

The first goal is **not sending CAN frames**.

It is simply verifying that the CPU can access the controller registers.

Example:

```c
printf("DCAN1 base: 0x%08lx\n", SOC_DCAN_1_REGS);
printf("CTL = 0x%08lx\n", REG32(SOC_DCAN_1_REGS + DCAN_CTL));
```

If register values can be read consistently, the MMIO mapping is working.

---

# Step 2. Enable Peripheral Clock

Many peripherals are inaccessible until their clocks are enabled.

Configure:

```text
CM_PER_DCAN1_CLKCTRL
```

Verify:

```text
MODULEMODE = ENABLE
IDLEST = Functional
```

Only after this step can the controller operate normally.

---

# Step 3. Configure Pin Multiplexing

Even if the controller is initialized correctly, communication will fail if the pins are still configured as UART or GPIO.

Configure:

```text
P9.24 -> DCAN1_TX
P9.26 -> DCAN1_RX
```

Verify pad configuration registers.

---

# Step 4. Enter Initialization Mode

Before modifying controller configuration:

```text
INIT = 1
CCE = 1
```

This allows:

- Bit timing configuration
- Message Object configuration

without active CAN communication.

---

# Step 5. Configure Bit Timing

Determine:

- Controller input clock
- Desired bitrate
- BRP
- TSEG1
- TSEG2

Verify:

```text
24 MHz
↓

125 kbps

↓

BTR = 0x00000517
```

---

# Step 6. Configure TX Message Object

Configure:

- Message ID
- DLC
- Data
- MSGVAL
- DIR

Transfer configuration through IF registers into Message Object memory.

Issue:

```text
TX Request
```

---

# Step 7. Validate Hardware Independently

Before debugging RTEMS software, validate the physical CAN path using Linux.

Topology:

```text
BBB Linux
    ↓
DCAN1
    ↓
CAN Transceiver
    ↓
CAN Bus
    ↓
CANable
    ↓
UTM Linux SocketCAN
```

Successfully receiving frames with `candump` confirms:

- Controller
- Transceiver
- Wiring
- Termination
- USB adapter
- SocketCAN

are all functioning correctly.

---

# Step 8. Validate RTEMS Transmission

Switch to RTEMS.

Topology:

```text
RTEMS BBB
    ↓
DCAN1
    ↓
CAN Bus
    ↓
CANable
    ↓
UTM Linux
```

Verify:

```bash
candump can0
```

Frame received:

```text
123 [8] 11 22 33 44 55 66 77 88
```

This proves the TX path is functional.

---

# Step 9. Configure RX Message Object

Create a receive Message Object.

Configure:

- MSGVAL
- RXIE
- UMASK
- ID filtering

Transfer configuration into Message Object memory.

Verify configuration by reading it back.

---

# Step 10. Validate RTEMS Reception

Send from Linux:

```bash
cansend can0 123#1122334455667788
```

Initially:

```text
NWDAT = 0
```

but

```text
RXOK = 1
```

Using RXOK as the trigger to force a Message Object read successfully returned:

```text
ID  = 0x123

DLC = 8

Data:
11 22 33 44 55 66 77 88
```

This confirmed that:

- CAN controller received the frame
- Message Object stored the frame correctly
- IF register transfer works
- Frame decoding works

---

# Step 11. Compare with Previous Driver

Study the previous implementation:

```text
dcan_init_rxobj()

dcan_read_obj()

dcan_save_msg()

dcan_isr()
```

instead of immediately integrating into the RTEMS CAN stack.

Understanding the receive flow first makes future integration much easier.

---

# Bring-up Philosophy

The bring-up process can be summarized as:

```text
Register Access
        ↓
Clock
        ↓
Pinmux
        ↓
Bit Timing
        ↓
TX Message Object
        ↓
Hardware Validation
        ↓
TX Functional
        ↓
RX Message Object
        ↓
RX Functional
        ↓
Interrupt Handling
        ↓
RTEMS CAN Stack Integration
```

Each layer builds upon the previous one.

If one layer is verified independently, debugging becomes significantly easier.

---

# Lessons Learned

A functional driver should not be developed by implementing everything at once.

Instead:

- verify hardware first,
- validate each register operation,
- prove TX independently,
- prove RX independently,
- then integrate with the operating system framework.

This incremental bring-up methodology greatly reduces debugging complexity and provides clear checkpoints throughout driver development.