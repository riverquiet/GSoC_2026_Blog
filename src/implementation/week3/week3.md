# week_3

# RTEMS AM335x D-CAN Driver Development (Weekly Progress)

## Overview

This week, the focus shifted from **register-level bring-up** to **driver architecture refactoring**. Instead of simply verifying that the AM335x D-CAN controller could transmit and receive CAN frames, I gradually reorganized the implementation to follow the **RTEMS CAN Framework** design by studying the **SJA1000 driver**.

The primary goal was:

> Transform a standalone D-CAN register test into a reusable RTEMS-style CAN driver skeleton.

---

# 1. Refactoring Register Test into Driver Structure

Originally, all initialization, register access, polling, and frame processing were implemented inside a single test routine.

The code was reorganized into multiple driver-oriented modules:

```
am335x_dcan_init_125k()

↓

am335x_dcan_process_interrupts()

↓

am335x_dcan_get_rx_frame()

↓

am335x_dcan_worker_like()

↓

dcan1_process_received_frame()
```

This modular structure is much closer to the organization of the official RTEMS CAN drivers.

---

# 2. Migration to RTEMS `struct can_frame`

The original implementation used a custom frame structure for received CAN messages.

During this week, the implementation was migrated to:

```c
struct can_frame
```

which is the standard frame representation used by the RTEMS CAN framework.

The receive path now becomes:

```
Linux CAN Frame
        ↓
DCAN Message Object
        ↓
struct can_frame
        ↓
Driver Internal State
        ↓
Worker-like Processing
        ↓
Upper Layer
```

Using the RTEMS data structure simplifies future framework integration and aligns the driver with the existing RTEMS CAN stack.

---

# 3. Driver Internal State

A dedicated driver private structure was introduced:

```c
struct am335x_dcan_internal
```

which currently stores:

- received CAN frame
- frame_to_pass flag
- RX callback
- callback argument
- worker semaphore
- RTEMS CAN chip pointer

instead of keeping these variables scattered throughout the implementation.

This organization follows the design philosophy of

```
struct sja1000_internal
```

from the official RTEMS SJA1000 driver.

---

# 4. Worker-like Processing Model

A polling-based worker was implemented:

```
worker_like()

↓

process_interrupts()

↓

get_rx_frame()

↓

process_received_frame()
```

instead of directly processing received frames inside the test application.

The worker continuously polls the controller interrupt register and dispatches received frames after they are converted into RTEMS CAN frames.

Although still polling-based, this design closely resembles the worker model used by the official RTEMS CAN drivers.

---

# 5. Callback Dispatch Layer

A simple callback mechanism was introduced:

```c
am335x_dcan_set_rx_callback()
```

allowing upper-layer code to register a receive handler.

Current receive flow:

```
worker_like()

↓

process_interrupts()

↓

internal frame

↓

callback(frame)

↓

test application prints frame
```

instead of

```
driver

↓

direct printf()
```

This callback layer serves as a temporary upper-layer interface and will eventually be replaced by the RTEMS CAN queue system.

---

# 6. Initial RTEMS Driver Skeleton

An initial

```c
rtems_can_am335x_dcan_initialize()
```

function was implemented.

Current responsibilities include:

```
allocate internal structure

↓

allocate rtems_can_chip

↓

allocate qends_dev

↓

chip->internal = internal

↓

internal->chip = chip

↓

return initialized chip
```

This implementation is modeled after

```
rtems_can_sja1000_initialize()
```

and represents the first step toward integrating the AM335x D-CAN driver into the RTEMS CAN framework.

---

# 7. Hardware Validation

The complete receive path has been successfully verified:

```
Linux

↓

cansend can0 123#1122334455667788

↓

CAN Bus

↓

AM335x DCAN Message Object 2

↓

worker_like()

↓

process_interrupts()

↓

struct can_frame

↓

callback()

↓

RTEMS prints:

RX ID=0x123
DLC=8
DATA=11 22 33 44 55 66 77 88
```

This confirms that the refactoring preserved the original hardware functionality while introducing a cleaner driver architecture.

---

# 8. Comparison with SJA1000 Driver

The development this week focused on gradually aligning the AM335x D-CAN driver with the architecture of the official RTEMS SJA1000 driver.

Current mapping:

| SJA1000 Driver | AM335x D-CAN Driver |
|----------------|---------------------|
| `struct sja1000_internal` | `struct am335x_dcan_internal` |
| `struct can_frame` | `struct can_frame` |
| `worker()` | `worker_like()` |
| `process_interrupts()` | `am335x_dcan_process_interrupts()` |
| `initialize()` | `rtems_can_am335x_dcan_initialize()` |
| queue dispatch | callback dispatch (temporary) |

Rather than focusing on hardware registers, the implementation is now evolving toward a framework-oriented RTEMS CAN driver.

---

# Current Progress

Completed:

- ✅ DCAN clock and pinmux configuration
- ✅ Bit timing configuration (125 kbps)
- ✅ TX Message Object configuration
- ✅ RX Message Object configuration
- ✅ Linux → RTEMS CAN communication
- ✅ Migration to RTEMS `struct can_frame`
- ✅ Driver internal state organization
- ✅ Worker-like processing model
- ✅ Callback dispatch layer
- ✅ Initial RTEMS CAN driver skeleton

---

# Next Steps

The next development stage will continue following the SJA1000 driver architecture.

Planned tasks include:

```
Register Wrapper

REG32(...)
        ↓
dcan_read_reg()
dcan_write_reg()

↓

internal->base

↓

RTEMS Queue Integration

↓

queue_filter_frame_to_edges()

↓

Application read()

↓

Full RTEMS CAN Driver
```

The overall development is gradually transitioning from a hardware validation project into a framework-compliant RTEMS CAN driver implementation.