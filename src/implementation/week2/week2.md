# Week 2: CAN Transmission Bring-up and Hardware Validation

**Period:** June 1 – June 5, 2026

## Overview

This week focused on moving from Message Object configuration toward actual CAN transmission bring-up on the AM335x D-CAN controller.

After successfully configuring and reading back a transmit Message Object, I began testing the transmission path. Although transmission requests could be issued, no CAN frames appeared on the external bus. To isolate the problem, I validated the hardware path under Linux, determined the correct controller clock frequency, and updated the CAN bit timing configuration.

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

Commands:

```bash
sudo ip link set can1 up type can bitrate 125000
sudo ip link set can0 up type can bitrate 125000

candump can0
cansend can1 123#1122334455667788
```

Result:

```text
can0 123 [8] 11 22 33 44 55 66 77 88
```

This confirmed that:

- The AM335x DCAN controller was functional
- The CAN transceiver was operating correctly
- Bus wiring and termination were correct
- The CANable adapter was functioning properly
- Linux SocketCAN was configured correctly

The successful test narrowed the remaining issue to the RTEMS software bring-up path.

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

## Summary

This week I:

- Added transmission request testing
- Validated the complete CAN hardware path using Linux SocketCAN
- Determined the AM335x system clock frequency
- Updated DCAN bit timing for 125 kbps operation
- Narrowed the remaining issue to the RTEMS software bring-up path

The next step is to continue investigating the RTEMS initialization sequence and controller configuration in order to achieve successful end-to-end CAN transmission.