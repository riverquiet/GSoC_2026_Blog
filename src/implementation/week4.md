# week_4 from bringup to RTEMS Driver framework

## Overview

This week, I continued developing the AM335x D-CAN driver on BeagleBone Black. The focus was to move from a simple register-level test toward a cleaner RTEMS CAN framework driver. Significant progress was made in initialization, bit timing support, register abstraction, interrupt handling, and worker task design.

---

## Driver Initialization Improvements

Several improvements were made to the initialization flow:

* Created and initialized the basic RTEMS CAN structures:

  * `chip`
  * `internal`
  * `qends_dev`
* Added worker task creation during initialization.
* Installed the DCAN interrupt handler.
* Cleaned up the startup sequence.
* Improved the overall driver structure to better follow existing RTEMS CAN drivers.

---

## RTEMS CAN Bit Timing Support

Previously, the bit timing parameters were hard-coded.

This week, the driver was converted to use the RTEMS CAN framework:

```text
bitrate
    ↓
rtems_can_bitrate2bittiming()
    ↓
bittiming structure
    ↓
dcan_set_bittiming()
    ↓
DCAN BTR register
```

### Completed

* Added `dcan_calc_and_set_bittiming()`.
* Used `rtems_can_bitrate2bittiming()` to calculate CAN timing parameters.
* Programmed the BTR register based on the calculated values.
* Removed hard-coded timing values.

This makes the driver more flexible and consistent with other RTEMS CAN drivers.

---

## Register Access Cleanup

Previously, many registers were accessed directly through:

```c
REG32(...)
```

These accesses were replaced with:

```c
dcan_read_reg()
dcan_write_reg()
```

### Benefits

* Cleaner code.
* Better encapsulation.
* Easier debugging.
* Less hardware-specific code scattered throughout the driver.

---

## Removing Global Variables

Private driver information is now stored inside:

```c
chip->internal
```

instead of using global variables.

### Benefits

* Better modularity.
* Easier future expansion.
* Similar structure to SJA1000 and CTU CAN FD drivers.

---

## Header and Register File Cleanup

Several code organization improvements were made:

* Reorganized register definitions.
* Cleaned up `dcan_regs.h`.
* Reduced dependency on legacy TI header files.
* Removed unnecessary include files.
* Began moving AM335x-specific definitions into local driver files.
* Simplified the code structure.

---

## Interrupt Handling

Following the mentor's recommendation, I studied the CTU CAN FD driver instead of SJA1000 because its interrupt architecture is simpler and closer to D-CAN.

### Interrupt Flow

```text
Hardware Interrupt
        ↓
ISR
        ↓
Disable interrupt
        ↓
Post worker semaphore
        ↓
Return immediately
```

The interrupt service routine remains short and avoids heavy processing.

### Worker Flow

```text
Worker wakes up
        ↓
Process interrupt source
        ↓
Read message object
        ↓
Deliver frame
        ↓
Re-enable interrupt
```

This design separates interrupt handling from frame processing and follows the CTU CAN FD style.

---

## Worker Task Skeleton

A real RTEMS worker task was implemented.

### Worker Responsibilities

* Wait on a binary semaphore.
* Process pending interrupts.
* Read received message objects.
* Deliver frames to the RTEMS CAN queue.
* Re-enable interrupts.

This replaces the earlier polling-based "worker-like" implementation.

---

## Receive Path Validation

Successfully verified the complete receive path:

```text
Linux SocketCAN
      ↓
CAN bus
      ↓
BBB DCAN1
      ↓
Hardware interrupt
      ↓
ISR
      ↓
Worker task
      ↓
Read RX message object
      ↓
RTEMS CAN queue
      ↓
RX callback
```

Multiple CAN frames were successfully received.

Example:

```text
RX ID=0x123 DLC=8 DATA=00 99 88 77 66 55 44 33
```

This confirms that the interrupt-driven receive mechanism is functioning correctly.

---

## Current Status

### Completed

* [x] Driver initialization skeleton
* [x] RTEMS CAN bit timing support
* [x] Register read/write abstraction
* [x] Remove global variables
* [x] Register file cleanup
* [x] Worker task skeleton
* [x] Interrupt handler installation
* [x] Interrupt-driven receive mechanism
* [x] Multiple-frame reception
* [x] Frame delivery to RTEMS CAN queue
* [x] Debug callback verification

---

## Architecture After This Week

```text
Linux SocketCAN
        ↓
CAN bus
        ↓
DCAN Controller
        ↓
Hardware Interrupt
        ↓
ISR
(disable interrupt + post semaphore)
        ↓
Worker Task
        ↓
dcan_process_interrupts()
        ↓
Read RX message object
        ↓
RTEMS CAN Queue
        ↓
Application / Callback
```

---

## Next Steps

### Code Cleanup

* Remove the old polling-based `worker_like()` implementation.
* Continue separating AM335x-specific definitions.
* Further simplify interrupt helper functions.

### RTEMS CAN Queue Study

* Study `can-queue.c`.
* Understand queue edges and queue slots.
* Learn how applications read frames from the queue.

### Application-Level Integration

Implement a queue-based application read path:

```text
Hardware
    ↓
Interrupt
    ↓
Worker
    ↓
RTEMS CAN Queue
    ↓
Application read()
```

### Transmit Path

Begin TX queue integration:

```text
Application
    ↓
RTEMS CAN Queue
    ↓
TX buffer
    ↓
DCAN Message Object
    ↓
CAN bus
```

### Long-Term Goal

Continue refining the driver toward full RTEMS CAN framework support and eventual upstream submission.
