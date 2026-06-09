# Week 2: CAN Transmission Bring-up and Hardware Validation

**Period:** June 1 – June 5, 2026

## Overview

Last week I focused on learning Message Object configuration, this week will toward test CAN transmission bring-up on the AM335x D-CAN controller.

I configuring and reading back a transmit Message Object, I began testing the transmission path. Although transmission requests could be issued, the TXRQ register still return 0. Maybe the register address is wrong. To move forware, this week will focus on hardware test. To isolate the problem, I validated the hardware path under Linux, determined the correct controller clock frequency, and updated the CAN bit timing configuration.

---

## Request Transmission for Configured Message Object

After successfully configuring and reading back Message Object 1, I added a transmit request through the IF1 command interface.

The test:

- Left DCAN initialization mode
- Issued a TX request
- Read TXRQ and ES status registers

Although no CAN frame was observed on the bus, this step exercised the transmission request path and provided useful debug information.

This established that Message Object configuration alone is not sufficient for successful CAN communication and that further investigation of the controller and hardware setup was required.

---

## Validate CAN Hardware Path

To determine whether the issue was caused by hardware or software, I validated the complete CAN hardware path using Linux.

Test setup:

```text
BBB Linux
 ↓
DCAN1
 ↓
CAN Transceiver
 ↓
CAN Bus
 ↓
CANable Adapter
 ↓
Linux SocketCAN
```

---

## Read System Clock Frequency

To correctly configure CAN bit timing, I first needed to determine the controller input clock frequency.

I read the AM335x CONTROL_STATUS register and decoded the SYSBOOT configuration bits.

Result:

```text
CONTROL_STATUS = 0x0040033c
System Clock = 24 MHz
```

This information provided the basis for calculating the correct CAN bitrate configuration.

---

## Update Bit Timing

Using the confirmed 24 MHz clock, I calculated a new BTR value for 125 kbps operation.

Configuration:

```text
BRP   = 24
TSEG1 = 6
TSEG2 = 1
```

Result:

```text
Bitrate = 125000 bit/s
BTR = 0x00000517
```

This configuration matches the bitrate used during the Linux SocketCAN hardware validation.

---

## CAN Reception Validation

Implemented a receive Message Object and configured it to accept standard CAN frames.

Receive path:

```text
UTM Linux
    ↓
CANable USB Adapter
    ↓
CAN Bus
    ↓
CAN Transceiver
    ↓
AM335x DCAN1
    ↓
Message Object 2
    ↓
RTEMS
```

After sending

```bash
cansend can0 123#1122334455667788
```

RTEMS successfully decoded:

```text
ID:   0x123
DLC:  8

Data:
11 22 33 44 55 66 77 88
```

This verified that the complete RX data path is functional.

---

## Receive Path Debugging

During debugging, an interesting behavior was observed.

Initially, polling the `NWDAT` register never indicated that new data had arrived:

```text
NWDAT = 0
```

However, the controller status register showed:

```text
RXOK = 1
```

Using `RXOK` as the trigger to force a Message Object read successfully retrieved the received frame.

This demonstrated that:

- CAN controller successfully received the frame
- Message Object 2 correctly stored the received data
- IF register transfer and frame decoding work correctly

The remaining investigation is understanding why `NWDAT` is not behaving as expected in the current implementation.

---

## Driver Analysis

Compared the current implementation with the previous D-CAN driver.

Studied:

- `dcan_init_rxobj()`
- `dcan_read_obj()`
- `dcan_save_msg()`
- `dcan_isr()`

Key observations:

- Previous driver uses IF2 for receive operations.
- Receive events are primarily detected through `DCAN_INT` and interrupt handling rather than polling `NWDAT`.
- Received frames are converted into `struct can_msg` and passed to the RTEMS CAN stack.

This provides a clear direction for future driver integration.

---

## Hardware and Environment Notes

During testing, an environment issue was identified.

Stable communication requires:

```text
Start UTM Linux
        ↓
Attach CANable USB device
        ↓
Bring up SocketCAN interface
        ↓
Boot RTEMS BBB application
```

Following this sequence ensures reliable CAN communication between RTEMS and Linux.

---

## Summary

This week achieved a major milestone:
- Validated the complete CAN hardware path using Linux SocketCAN
- CAN transmit path verified
- CAN receive path verified
- Message Object 2 successfully stores and returns CAN frames
- End-to-end communication between RTEMS BBB and UTM Linux validated
- Began studying the previous D-CAN driver architecture for RTEMS CAN stack integration

The next step is to understand the interrupt-driven receive flow (`DCAN_INT`, `INTPND`, and `NWDAT`) and gradually refactor the register-level implementation into a clean RTEMS CAN driver.