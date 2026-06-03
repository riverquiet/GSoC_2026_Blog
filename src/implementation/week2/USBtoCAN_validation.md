# Linux SocketCAN Hardware Validation for AM335x DCAN1 (BeagleBone Black Linux → UTM Linux via CANable)

## Objective

Before continuing RTEMS D-CAN driver bring-up, validate that the external CAN hardware path is functioning correctly.

The goal is to determine whether communication issues originate from hardware or from the RTEMS software stack.

---

## Test Environment

### Transmitter

- BeagleBone Black
- Debian Linux
- AM335x DCAN1 controller
- External CAN transceiver

### Receiver

- UTM Virtual Machine
- Debian Linux
- CANable USB-to-CAN adapter
- Linux SocketCAN tools

---

## Hardware Setup

### BeagleBone Black

DCAN1 interface:

```text
P9.24 -> DCAN1_TX
P9.26 -> DCAN1_RX
```

Connected to an external CAN transceiver.

### CANable USB-to-CAN Adapter

Connections:

```text
CANH -> CANH
CANL -> CANL
GND  -> GND
```

Bus termination:

```text
120Ω resistor at one end
120Ω resistor at the other end
```

CANable termination switch:

```text
OFF
```

---

## Validation Topology

```text
BBB Linux
    ↓
AM335x DCAN1
    ↓
CAN Transceiver
    ↓
CAN Bus
    ↓
CANable USB-to-CAN Adapter
    ↓
UTM Linux VM
    ↓
Linux SocketCAN
    ↓
candump
```

---

## Connecting CANable to UTM Linux

Initially, the CANable device did not appear inside the Linux virtual machine.

Checking available interfaces:

```bash
ip link
```

No CAN interface was present.

### Root Cause

The USB device was attached to macOS instead of the UTM virtual machine.

### Solution

In UTM:

```text
UTM
 └─ USB Devices
     └─ Connect
         └─ canable gs_usb
```

After attaching the device to the VM:

```bash
ip link
```

Result:

```text
can0
```

The CANable adapter was successfully detected by Linux.

---

## Configure BBB Linux

Configure pin multiplexing:

```bash
config-pin p9.24 can
config-pin p9.26 can
```

Enable CAN interface:

```bash
sudo ip link set can1 up type can bitrate 125000
```

Verify:

```bash
ip -details link show can1
```

---

## Configure UTM Linux SocketCAN

Bring up the CANable interface:

```bash
sudo ip link set can0 up type can bitrate 125000
```

Verify:

```bash
ip -details link show can0
```

---

## Monitor CAN Traffic

Start monitoring CAN frames:

```bash
candump can0
```

The terminal waits for incoming frames.

---

## Transmit Test Frame

From BBB Linux:

```bash
cansend can1 123#1122334455667788
```

Frame contents:

```text
ID  = 0x123
DLC = 8

11 22 33 44 55 66 77 88
```

---

## Observed Result

UTM Linux successfully received:

```text
can0 123 [8] 11 22 33 44 55 66 77 88
```

---

## Validation Outcome

The successful transmission confirms:

- AM335x DCAN1 controller is operational
- External CAN transceiver is operational
- CAN bus wiring is correct
- Bus termination is correct
- CANable adapter is operational
- UTM USB passthrough is functioning correctly
- Linux SocketCAN configuration is correct
- 125 kbps communication is working

---

## Engineering Conclusion

Because Linux successfully transmitted and received CAN frames through the same hardware path, the physical CAN network was verified to be working correctly.

This allowed subsequent debugging efforts to focus exclusively on the RTEMS software bring-up path rather than hardware issues.

The validation significantly reduced the debugging scope and ultimately helped identify a missing DCAN1 pin multiplexing configuration in the RTEMS test application.

---

## Key Lesson

When performing embedded driver bring-up, validating the hardware path independently can save significant debugging time.

By proving that the CAN controller, transceiver, bus, CANable adapter, UTM USB passthrough, and SocketCAN tools all work correctly under Linux, remaining failures can be isolated to software configuration issues rather than hardware faults.