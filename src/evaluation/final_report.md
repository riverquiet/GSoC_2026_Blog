# GSoC 2026 Final Report: Bosch DCAN Driver for RTEMS

**Organization:** RTEMS\
**Contributor:** Ning Zhang\
**Project:** Bosch DCAN Driver for RTEMS on BeagleBone Black\
**Platform:** BeagleBone Black (TI AM335x)\
**RTOS:** RTEMS 7

## 1. Project Overview

The goal of this Google Summer of Code project was to develop Bosch DCAN
controller support for RTEMS and validate it on the BeagleBone Black.

The TI AM335x processor contains DCAN controllers based on the Bosch CAN
architecture. The project required both low-level hardware support and
integration with the RTEMS CAN framework.

The work started with register-level bring-up and gradually moved toward
an RTEMS driver architecture with interrupts, queues, error handling, RX
FIFO support, TX completion, and controller lifecycle management.

## 2. Development and Test Environment

-   RTEMS 7.
-   BeagleBone Black with TI AM335x.
-   Bosch DCAN controller.
-   Linux SocketCAN.
-   CANable USB-to-CAN adapter.
-   Physical CAN bus with proper termination.
-   125 kbit/s CAN bitrate for the main driver tests.

Linux `cansend` and `candump` were useful during early hardware
validation. Later tests also used both DCAN controllers on the
BeagleBone Black.

## 3. Work Completed

### 3.1 Basic DCAN Bring-up

The first stage included enabling the DCAN clock, configuring AM335x
pinmux, accessing DCAN registers, configuring bit timing and Message
Objects, and testing CAN TX and RX.

Successful bidirectional communication confirmed the basic hardware
configuration.

### 3.2 RTEMS CAN Framework Integration

The driver was integrated with the RTEMS CAN framework instead of
keeping an application-specific frame path.

This work included RTEMS CAN frame structures, CAN device registration,
TX/RX queues, TX queue-slot ownership, local echo, and CAN statistics.

### 3.3 Interrupt and Worker Design

The driver uses a short interrupt handler and performs most processing
in a worker task. The worker processes controller status, RX events, TX
completion, and pending TX work before interrupts are re-enabled.

### 3.4 RX FIFO

Multiple DCAN Message Objects are used for receive processing. The
implementation checks hardware state including `NewDat`, `MsgLst`, and
end-of-buffer information.

The driver reports `MsgLst` as an RX overflow through the RTEMS CAN
framework and statistics.

### 3.5 Error Handling

The driver handles controller states including error active, error
warning, error passive, and bus off. Protocol errors such as ACK, bit,
stuff, form, and CRC errors are also reported where appropriate.

### 3.6 TX Queue and Completion Handling

The TX path obtains frames from RTEMS CAN queues and keeps ownership of
a queue slot until transmission completes or the controller is stopped.

After completion, the driver updates statistics, provides local echo
when required, releases the queue slot, and continues pending TX work.

The controller-stop path also cleans up pending transmissions.

### 3.7 DCAN0 and DCAN1

The project initially focused on DCAN1. Later, support was extended so
that DCAN0 and DCAN1 could both be registered and tested.

## 4. TX Ordering Investigation

TX scheduling was one of the most challenging parts of the project.

The initial driver design did not implement TX preemption or Message
Object rearrangement. When I extended the TX path to use multiple
hardware TX Message Objects, I found that simply filling several free
Message Objects from the RTEMS software queue did not guarantee that
the transmission order would match the software queue order.

For example, in a same-priority test, 64 frames were queued in the
order:

'''text
0 1 2 3 4 5 6 7 ...
'''

With multiple TX Message Objects active, the received order included
patterns such as:

'''text
0 2 1 3
4 6 5 7
8 10 9 11
'''

This showed that assigning frames to multiple hardware Message Objects
was not enough to preserve FIFO ordering.

I then experimented with a more advanced TX scheduling design. The idea
was to track the RTEMS queue priority associated with each hardware TX
Message Object and rearrange pending Message Objects when a
higher-priority frame arrived.

Some small tests showed the expected ordering, but larger tests exposed
stability problems, including reordered or duplicated frames. This
indicated that aborting, reassigning, and restoring active TX Message
Objects requires more careful synchronization and ownership handling.

Because this experimental preemption code was not stable enough for
upstream submission, it is not part of the current upstream driver.

For the current baseline, the driver intentionally uses one hardware TX
Message Object:

'''text
RTEMS TX queue
      |
      v
    MO9
      |
      v
   CAN bus
'''

This keeps the TX path simple and guarantees that frames from the same
priority class are transmitted in FIFO order.

I verified this behavior with a same-priority test containing 64 frames.
All frames used the same CAN ID and queue priority, with sequence numbers
from 0 to 63.

'''text
Expected frames:  64
Received frames:  64
Order errors:     0
Invalid frames:   0
CAN error frames: 0
Verified order:   0..63
'''

The test passed.

Support for multiple TX Message Objects with priority-aware scheduling
remains future work. The planned direction is to use
`rtems_can_queue_pending_outslot_prio()` and rearrange TX Message Objects
when a higher-priority class would otherwise be blocked by a
lower-priority one.


## 5. Code Organization

Generic Bosch DCAN controller logic is separated from AM335x-specific
support such as peripheral clock and pinmux configuration. This
organization was improved based on mentor and merge-request feedback.

## 6. Upstream Work

The driver has been developed through the RTEMS GitLab review process.

-   **Merge request:** \[Add RTEMS GitLab merge request URL here\]
-   **Final GSoC commit:** \[Add final GSoC commit URL here\]
-   **Relevant source files:** \[Add final source links here\]

These links should be updated before this report is submitted as the
official GSoC work product.

## 7. Challenges and Lessons Learned

### Understanding DCAN

DCAN uses hardware Message Objects and interface registers.
Understanding how these parts interact was a major part of the project.

### Hardware Testing

Real hardware testing was essential. Testing with the BeagleBone Black,
CANable, and Linux SocketCAN helped make controller behavior visible.

### RTEMS Integration

A working hardware prototype is different from an upstream-quality RTEMS
driver. Queue ownership, synchronization, error handling, code
organization, and lifecycle management are important.

### TX Scheduling

The multi-TX-Message-Object experiments showed that more hardware
parallelism also creates more scheduling complexity. The driver must
consider both RTEMS software queues and DCAN hardware behavior.

### Open-Source Development

Mentor review helped improve the architecture and code quality. I
learned to keep changes reviewable, test changes on hardware, manage Git
history, and separate generic code from board-specific code.

## 8. Remaining Work

The main future improvement is priority-aware multi-Message-Object TX
scheduling.

The planned direction is:

-   Support multiple TX Message Objects.
-   Use `rtems_can_queue_pending_outslot_prio()` to inspect pending
    queue priority.
-   Prevent a higher-priority class from being blocked by lower-priority
    frames already assigned to hardware TX Message Objects.
-   Rearrange TX Message Objects while preserving FIFO ordering within
    the same priority class.
-   Add focused tests for priority ordering and TX Message Object
    rearrangement.

Additional cleanup may also be required through normal RTEMS upstream
review.

## 9. Final Status

At the end of GSoC, the project has a functional Bosch DCAN driver
running on RTEMS 7 and tested on BeagleBone Black hardware.

The project progressed from basic register-level CAN communication to
RTEMS CAN framework integration, including interrupt-driven receive, RX
FIFO handling, TX queue integration, statistics, error reporting,
controller lifecycle handling, and DCAN0/DCAN1 support.

The current TX baseline uses one hardware TX Message Object and has
verified same-priority FIFO ordering. More advanced priority-aware
multi-Message-Object TX scheduling is left as future work.

## 10. Links

-   **GSoC project:** \[Add project URL\]
-   **RTEMS merge request:** \[Add merge request URL\]
-   **Final GSoC commit:** \[Add commit URL\]
-   **Test notes / blog posts:** \[Add links if available\]

## 11. Acknowledgements

Thank you to my RTEMS mentor and the RTEMS community for the technical
guidance, code review, and support throughout Google Summer of Code
2026.
