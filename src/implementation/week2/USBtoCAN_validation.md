# Linux SocketCAN Hardware Validation for AM335x DCAN1

## Objective

Before continuing RTEMS D-CAN driver bring-up, validate that the external CAN hardware path is functioning correctly.

The goal is to determine whether communication issues originate from hardware or from the RTEMS software stack.

---

## Hardware Setup

### BeagleBone Black

DCAN1 interface:

```text
P9.24 -> DCAN1_TX
P9.26 -> DCAN1_RX
```

Connected to an external CAN transceiver.

### USB-to-CAN Adapter

CANable adapter connected to a Linux virtual machine.

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
Linux SocketCAN
    ↓
candump
```

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

Verify interface:

```bash
ip -details link show can1
```

---

## Configure Linux SocketCAN

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

Start CAN frame monitoring:

```bash
candump can0
```

Terminal waits for incoming frames.

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

Linux SocketCAN successfully received:

```text
can0  123   [8]  11 22 33 44 55 66 77 88
```

---

## Validation Outcome

The successful transmission confirms:

- AM335x DCAN1 controller is operational
- External CAN transceiver is operational
- CAN bus wiring is correct
- Bus termination is correct
- CANable adapter is operational
- USB passthrough is functioning correctly
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

By proving that the CAN controller, transceiver, bus, and monitoring tools work correctly under Linux, remaining failures can be isolated to software configuration issues rather than hardware faults.