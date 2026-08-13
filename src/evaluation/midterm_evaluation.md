# GSoC 2026 Midterm Evaluation

## Project

**Organization:** RTEMS\
**Project:** Bosch DCAN Driver for RTEMS on BeagleBone Black\
**Contributor:** Ning Zhang

## Project Progress at Midterm

At the midterm, I had completed the basic bring-up and hardware
validation of the Bosch DCAN controller on the BeagleBone Black
(AM335x).

The main work completed at this stage included:

-   DCAN clock and pin configuration on the BeagleBone Black.
-   Basic DCAN register configuration.
-   CAN transmission from RTEMS to Linux.
-   CAN reception from Linux to RTEMS.
-   Interrupt and `DCAN_INT` testing.
-   Initial integration with the RTEMS CAN framework.
-   Building the driver with RTEMS and checking the code with warnings
    treated as errors.
-   Testing the driver on real BeagleBone Black hardware.

My hardware test setup used RTEMS 7 on the BeagleBone Black and Linux
SocketCAN with a CANable adapter. The CAN bus was tested at 125 kbit/s.
Basic TX and RX communication worked successfully.

I also opened and continued working on the RTEMS merge request. Based on
mentor feedback, I started improving the code structure. An important
change was separating the generic Bosch DCAN driver code from
AM335x-specific clock and pinmux code.

## Communication With My Mentor

I communicated with my mentor regularly through the RTEMS development
and review process. The merge request was kept as a draft while the
implementation was still being developed.

The mentor feedback helped me understand not only how to make the
hardware work, but also how a driver should be designed for upstream
RTEMS.

## Most Challenging Part So Far

The most challenging part was understanding the Bosch DCAN hardware and
connecting its Message Object model with the RTEMS CAN framework.

At first, much of the work was low-level hardware bring-up: clocks,
pinmux, registers, Message Objects, and interrupts. After basic
communication worked, the challenge became designing the driver in a way
that fits the RTEMS CAN stack.

I also learned that hardware functionality alone is not enough for an
upstream driver. Code organization, generic interfaces, error handling,
synchronization, and maintainability are also important.

## What I Planned to Work on After Midterm

-   Improve RTEMS CAN stack integration.
-   Use the RTEMS `struct can_frame` throughout the driver.
-   Improve the interrupt and worker-task design.
-   Implement RX FIFO handling with multiple Message Objects.
-   Improve TX completion handling.
-   Add CAN error reporting and statistics.
-   Improve controller start/stop behavior.
-   Continue hardware testing.
-   Respond to merge-request review comments and prepare the driver for
    upstream review.

## Midterm Reflection

I was happy that the basic DCAN communication was working on real
hardware by the midterm. The project also gave me a much better
understanding of RTEMS driver development and CAN controller hardware.

The next stage was more about making the implementation robust and suitable for
the RTEMS CAN framework.
