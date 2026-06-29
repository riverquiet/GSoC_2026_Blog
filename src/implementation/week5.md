# week_5

## Overview

This week I mainly worked on two parts of the D-CAN driver:

1. Debugging the **IF3 receive path** and understanding how it works.
2. Integrating the **TX path into the RTEMS CAN framework worker**, so transmission follows the standard RTEMS CAN architecture.

I also spent time cleaning the driver source code to prepare for the first merge request.

---

# IF3 Debugging

## Goal

My goal was to understand how the IF3 register works on the AM335x D-CAN controller.

Unlike IF1 and IF2, IF3 is designed as an **automatic update interface**. According to the AM335x Technical Reference Manual, IF3 can automatically copy the content of a selected Message Object into the IF3 registers.

I wanted to verify whether IF3 could become another receive path in the driver.

---

## Initial Design

I configured two receive Message Objects.

- **Message Object 2**
  - ID = 0x123
  - Normal interrupt receive using IF1

- **Message Object 64**
  - ID = 0x153
  - IF3 automatic update

Then I enabled the IF3UPD register and periodically read:

- IF3ARB
- IF3MCTL
- IF3DATA
- IF3DATB

The first test was successful. IF3 could receive CAN frames correctly when tested alone.

---

## Problem

The problem appeared after I enabled interrupts.

The IF1 interrupt receive path continued working correctly, but IF3 stopped updating.

At first, I thought the problem was related to:

- IF3 configuration
- IF3UPD register
- IF3OBS register
- DE3 bit

I spent a long time checking these registers one by one, but none of them solved the problem.

---

## Debug Process

To better understand the hardware behavior, I added many debug messages.

I printed several registers before and after reading IF3:

- DCAN_INT
- ES
- INTPND1
- INTPND2
- NWDAT1
- NWDAT2

These debug messages helped me understand exactly what happened after each received CAN frame.

---

## Root Cause

Finally, I found that the problem was **not IF3 itself**.

The real reason was that **Message Object 64 was generating an interrupt**.

When the interrupt occurred:

1. The worker thread woke up.
2. The worker processed the interrupt.
3. IF1 read the Message Object.
4. The interrupt pending bit and NewDat bit were cleared.

After that, IF3 no longer had a new frame to update.

In other words, **IF1 and IF3 were trying to access the same Message Object**, causing them to interfere with each other.

---

## Final Solution

The solution was surprisingly simple.

I removed the **RXIE** bit from Message Object 64.

The final design became:

**Message Object 2**

- RX interrupt enabled
- Used by the RTEMS receive queue

**Message Object 64**

- RX interrupt disabled
- Used only for IF3 automatic update

Now the two receive paths are completely independent.

Both paths work correctly at the same time.

---

## Another Problem

At first, IF3 could only receive one CAN frame.

The second CAN frame never appeared.

I first suspected the NewDat bit and tried clearing it manually.

However, after adding more debug messages, I realized that NewDat was not the problem.

The real issue was my polling logic.

IF3 automatically keeps the latest received frame in its registers.

My polling loop kept reading the same data repeatedly, so it looked like no new frames were arriving.

I changed the polling logic so that only newly updated data is processed.

After this change, IF3 successfully received multiple CAN frames.

---

## What I Learned

This debugging process helped me understand the D-CAN receive architecture much better.

The most important conclusion is:

**IF3 is not designed to use interrupts.**

Instead, IF3 is designed for automatic register updates and is intended to work with polling or DMA.

Normal interrupt-driven receive should continue using IF1.

---

# TX Path Integration into the RTEMS CAN Framework

## Previous Implementation

Previously, the driver transmitted CAN frames directly.

The driver configured Message Object 1 and immediately requested transmission.

Although this verified the hardware, it did not follow the RTEMS CAN framework design.

---

## New Goal

The goal was to make transmission follow the standard RTEMS CAN architecture.

The new flow is:

```text
Application
        ↓
write("/dev/can0")
        ↓
RTEMS CAN TX Queue
        ↓
Worker Thread
        ↓
DCAN Driver
        ↓
Message Object 1
        ↓
CAN Bus
```

Instead of sending frames directly, the application simply writes a CAN frame into `/dev/can0`.

The RTEMS CAN framework stores the frame inside the TX queue.

The worker thread later retrieves one frame from the queue and sends it through the hardware.

---

## Implementation

To implement this feature, I studied the **SJA1000** driver because it is the reference CAN driver inside RTEMS.

I followed the same design.

The worker now performs the following steps:

1. Check whether the hardware TX Message Object is available.
2. Get one transmit slot from the RTEMS TX queue.
3. Copy the CAN frame into Message Object 1.
4. Request hardware transmission.
5. Notify the RTEMS CAN framework after transmission is completed.

This makes the D-CAN driver consistent with the RTEMS CAN framework.

---

## Testing

I created a simple application writer task.

The application opens:

```c
/dev/can0
```

and sends a CAN frame using:

```c
write(fd, &tx, sizeof(tx));
```

instead of calling hardware functions directly.

Linux SocketCAN successfully received:

```text
ID = 0x333
DATA = 11 22 33 44 55 88 88 88
```

This verified that the complete software stack works correctly.

---

## What I Learned

This work helped me understand the architecture of the RTEMS CAN framework.

The application never communicates directly with the hardware.

Instead, it only sends CAN frames into the framework queue.

The worker thread is responsible for retrieving frames from the queue and transmitting them through the hardware.

This design separates the application layer from the hardware layer and makes the driver much cleaner.

---

# Code Cleanup

Besides implementing new features, I also started cleaning the driver source code for the first merge request.

The cleanup included:

- Removing unnecessary forward function declarations.
- Reordering functions so that most forward declarations are no longer needed.
- Deleting unused helper functions.
- Removing duplicated code.
- Cleaning old debugging comments.
- Organizing the file structure to make it easier to read.

This cleanup makes the driver closer to the coding style used in other RTEMS drivers.

---

# Plans for Next Week

Next week I plan to:

- Continue cleaning the driver for the first merge request.
- Follow the RTEMS coding style more closely.
- Remove temporary debugging code.
- Improve TX and RX error handling.
- Continue studying whether IF3 can eventually deliver received frames into the RTEMS CAN queue using automatic update or DMA instead of interrupts.

Overall, this week I made good progress in understanding both the AM335x D-CAN hardware and the RTEMS CAN framework. The driver now supports a complete TX path through the RTEMS CAN queue, and I also gained a much deeper understanding of how IF3 works and why it should not be treated as a normal interrupt-driven receive path.