# GSoC 2026 Final Evaluation

## Project

**Organization:** RTEMS\
**Project:** Bosch DCAN Driver for RTEMS on BeagleBone Black\
**Contributor:** Ning Zhang

## Link to My GSoC Work

**Final work product:** [Local Repo](https://gitlab.rtems.org/River/rtems-bbb-dcan)\
**RTEMS merge request:** [RTEMS GitLab merge request](https://gitlab.rtems.org/rtems/rtos/rtems/-/merge_requests/1341)

## What I Completed

During GSoC 2026, I developed and tested a Bosch DCAN driver for RTEMS 7
using the BeagleBone Black AM335x platform.

The main completed work includes:

-   Bosch DCAN controller support for RTEMS.
-   AM335x BeagleBone Black clock and pinmux support.
-   DCAN0 and DCAN1 registration and configuration.
-   Integration with the RTEMS CAN framework.
-   CAN frame transmission and reception.
-   Interrupt-driven receive handling.
-   Worker-task processing outside the interrupt handler.
-   RX FIFO handling using multiple DCAN Message Objects.
-   Detection of the DCAN `MsgLst` condition and RX overflow reporting.
-   TX queue integration and queue-slot ownership handling.
-   TX completion handling and local echo.
-   CAN TX/RX statistics.
-   CAN controller error-state and protocol-error reporting.
-   Controller start and stop synchronization.
-   Cleanup of pending TX frames when stopping the controller.
-   Hardware testing with Linux SocketCAN and a CANable adapter.

The main hardware tests used a CAN bitrate of 125 kbit/s.

## TX Ordering

TX scheduling was one of the most challenging parts of the project.

I first extended the TX path to use multiple hardware TX Message Objects.
However, testing showed that simply filling multiple Message Objects did
not guarantee that the transmission order would match the RTEMS software
queue order.

I also experimented with priority-aware TX scheduling and Message Object
rearrangement. Some small tests showed the expected ordering, but larger
tests exposed stability problems, including reordered or duplicated
frames.

Because this experimental implementation was not stable enough for
upstream submission, it is not included in the current upstream driver.

For the current baseline, I use one hardware TX Message Object. This keeps
the TX path simple and preserves FIFO ordering within the same RTEMS CAN
queue priority class.

I verified this with a 64-frame same-priority test. All frames used the
same CAN ID and queue priority, with sequence numbers from 0 to 63.

The result was:

- 64 frames sent and received.
- 0 ordering errors.
- 0 invalid frames.
- 0 CAN error frames.
- Received sequence: `0..63`.

The test passed.

Support for multiple TX Message Objects with priority-aware scheduling
remains future work.

## Most Challenging Parts

The most challenging parts were understanding the DCAN Message Object
architecture, connecting DCAN hardware behavior with the RTEMS CAN queue
model, designing interrupt and worker processing, handling RX FIFO
overflow and CAN errors, managing TX queue-slot ownership, and
preserving TX ordering.

These problems helped me understand that driver development is not only
register programming. Synchronization, queue ownership, error paths,
hardware behavior, and maintainable architecture are equally important.

## Current State and Remaining Work

The driver can communicate on real BeagleBone Black hardware and is
integrated with the RTEMS CAN framework.

The current upstream baseline intentionally uses one TX Message Object.
This provides a simple and stable TX path and preserves FIFO ordering
within the same priority class.

The main remaining TX improvement is support for multiple TX Message
Objects with priority-aware scheduling. The planned direction is to use
`rtems_can_queue_pending_outslot_prio()` and rearrange TX Message Objects
when a higher-priority class would otherwise be blocked by a
lower-priority one.

I experimented with this design during GSoC, but the experimental version
was not stable enough to include in the upstream driver. Therefore, this
work is left as a TODO instead of adding unstable code to the final
baseline.

Additional changes may also come from the normal RTEMS upstream review
process and further hardware testing.

## GSoC Experience

My favorite part of GSoC was seeing a low-level driver progress from
register-level experiments to real CAN communication and then to
integration with the RTEMS CAN framework.

The project improved my skills in embedded systems, RTOS driver
development, CAN bus hardware, interrupt handling, debugging on real
hardware, Git, code review, and open-source development.

I plan to continue contributing to RTEMS and improving the driver based
on upstream feedback.