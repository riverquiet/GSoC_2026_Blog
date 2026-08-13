# DCAN TX Priority and Preemption Experiments

## Overview

After the basic DCAN TX and RX paths were working, I started to improve
the TX side of the driver.

The first working version used only one TX Message Object. This design
was simple because one RTEMS CAN queue slot was connected to one hardware
TX Message Object.

Later, I extended the driver to use multiple TX Message Objects. I used:

- MO1-MO8 for RX
- MO9-MO12 for TX

The goal was to improve TX performance and also study how TX priority
should work when several CAN frames are already waiting inside the DCAN
hardware.

This became more complicated than I expected, but the experiments helped
me understand the relationship between the RTEMS CAN queue, DCAN Message
Objects, and CAN bus arbitration.


## 1. Multi-TX Message Objects

With four TX Message Objects, the basic idea is:

    RTEMS CAN TX queue
            |
            v
       DCAN worker
            |
            v
    +-------+-------+-------+-------+
    |  MO9  | MO10  | MO11  | MO12 |
    +-------+-------+-------+-------+

Each active Message Object keeps one RTEMS queue slot:

    MO9  <-> slot A
    MO10 <-> slot B
    MO11 <-> slot C
    MO12 <-> slot D

This gives an important ownership rule:

> One RTEMS CAN queue slot should be owned by only one TX Message Object
> at one time.

When a Message Object finishes transmission, the driver returns a local
echo, updates the TX statistics, and releases the related queue slot.


## 2. Why TX Priority Is More Complicated

CAN already has hardware arbitration.

For normal CAN frames:

> A lower CAN identifier has higher arbitration priority.

For example:

    CAN ID 0x020 -> high priority
    CAN ID 0x500 -> medium priority
    CAN ID 0x700 -> low priority

However, RTEMS also has software TX queues with their own priorities.

This means there are several priority levels:

    Application
        |
        v
    RTEMS software TX queue priority
        |
        v
    DCAN TX Message Objects
        |
        v
    CAN ID hardware arbitration
        |
        v
    CAN bus

A high-priority frame may enter the RTEMS queue after several
lower-priority frames have already been copied into hardware Message
Objects.

This was the reason I started experimenting with TX preemption.


## 3. First Preemption Idea

Suppose all four TX Message Objects are busy:

    MO9   -> low-priority frame
    MO10  -> low-priority frame
    MO11  -> low-priority frame
    MO12  -> low-priority frame

Then a new high-priority frame arrives.

The idea was:

    high-priority frame enters software queue
                    |
                    v
           all TX MOs are busy
                    |
                    v
       find a low-priority TX frame
                    |
                    v
          cancel its TxRqst
                    |
                    v
      return old frame to software queue
                    |
                    v
       send high-priority frame first

This idea was partly inspired by the SJA1000 driver, which can abort a
pending hardware transmission when a higher-priority software frame is
waiting.


## 4. CAN-ID Based Hardware Reordering

I later tested a more complete design.

The idea was to keep the active DCAN TX Message Objects ordered by CAN ID.

For example:

    MO9  -> ID 54
    MO10 -> ID 55
    MO11 -> ID 57
    MO12 -> ID 60

Now a new frame with ID 56 arrives.

Because ID 56 has higher CAN priority than ID 57 and ID 60, the desired
hardware order becomes:

    MO9  -> ID 54
    MO10 -> ID 55
    MO11 -> ID 56
    MO12 -> ID 57

and ID 60 is returned to the software queue.

So the driver needs to:

1. Find the correct insertion position.
2. Cancel the affected hardware TX requests.
3. Wait for TxRqst to clear.
4. Move the software ownership information.
5. Insert the new frame.
6. Return the displaced frame to the RTEMS queue.
7. Rewrite the affected DCAN Message Objects.

I created several tests for front, middle, and tail insertion.

These controlled tests showed that the basic CAN-ID ordering idea can
work.


## 5. TX Compaction

Another problem happens when Message Objects finish independently.

For example:

    MO9  -> free
    MO10 -> ID 54
    MO11 -> ID 55
    MO12 -> ID 56

Now there is a hole in the hardware TX window:

    [ free ][ 54 ][ 55 ][ 56 ]

I experimented with a compaction operation:

    before:

    [ NULL, 54, 55, 56 ]

    after:

    [ 54, 55, 56, NULL ]

Then the driver can take another frame from the RTEMS TX queue and put it
into the last free Message Object.

This made the TX scheduling design more complete, but also much more
complex.


## 6. A Timing-Sensitive Bug

During these experiments, I found an interesting problem.

When the driver contained many printf() debug messages, the preemption
code sometimes appeared to work correctly.

After removing the debug prints, the driver became much faster, but some
failures appeared.

I observed errors such as:

    DUPLICATE FROM QUEUE

    queue returned owned slot

    SLOT HAS MULTIPLE OWNERS

    FIFO self-loop risk

This was an important clue.

The printf() calls were not fixing the driver. They were changing the
timing of the system.

The debug output slowed down:

- the worker task
- TX completion processing
- queue operations
- hardware Message Object updates

Because of this, some timing-sensitive problems became harder to
reproduce.

This is similar to a Heisenbug: adding debug code changes the timing and
changes the behavior of the bug.


## 7. Queue Slot Ownership Became the Main Debugging Question

At first, I thought the main problem was the preemption algorithm.

Later, I realized that the more important question was:

> Who owns each RTEMS CAN queue slot at each point in time?

The expected ownership lifecycle is:

    RTEMS queue owns slot
            |
            | test_outslot()
            v
    DCAN driver owns slot
            |
            | hardware transmission
            v
    TX completion
            |
            | free_outslot()
            v
    RTEMS queue owns slot again

A slot should never be owned by two TX Message Objects at the same time.

It should also not be returned by the RTEMS queue while the driver still
uses it.

I added ownership checks and a small in-memory trace to record operations
such as:

    GET
    MOVE
    REQUEUE
    FREE

This helped show that some failures were related to queue-slot lifetime,
not only to CAN-ID preemption.


## 8. A/B Test

To understand the problem better, I disabled runtime priority preemption
but kept the multi-TX-MO implementation.

The duplicate ownership problem could still appear.

This was an important result because it showed:

> The problem was not caused only by the preemption algorithm.

Preemption and compaction made the TX path more complicated and made the
problem easier to reproduce, but the deeper issue involved multiple
outstanding RTEMS TX queue slots and their ownership lifetime.


## 9. Returning to a Simpler Baseline

Because of these results, I decided not to use the complex preemption
implementation as the main upstream driver design.

The current preferred design is simpler:

    RTEMS TX queue
          |
          v
    find free TX MO
          |
          v
    one slot <-> one MO
          |
          v
    hardware transmission
          |
          v
    completion of that MO
          |
          v
    local echo + free slot

The current TX Message Objects are independent:

    MO9  <-> slot A
    MO10 <-> slot B
    MO11 <-> slot C
    MO12 <-> slot D

The completion of MO11 does not need to wait for MO9 or MO10.

I also separated the DCAN interface registers:

    IF1 -> RX and deferred RX/interrupt processing
    IF2 -> TX

This keeps the design simple while still using an idea from the Linux
C_CAN/DCAN driver that is useful for this hardware.


## 10. Current Latency Test Topology

The current latency test uses both AM335x DCAN controllers.

The topology is:

    DCAN0 (/dev/can0)
        |
        +-- high-priority sender
        |      CAN ID = 0x020
        |
        +-- low-priority sender
               CAN ID = 0x700


    DCAN1 (/dev/can1)
        |
        +-- medium-priority sender
        |      CAN ID = 0x500
        |
        +-- receiver

All controllers are connected to the same physical CAN bus.

The receiver is mainly used to measure the high-priority ID 0x020
traffic.

The test also uses paced traffic instead of unlimited write loops.

For example:

    send 4 frames
        |
        v
    wait 10 ms
        |
        v
    send next 4 frames

This makes the workload more balanced and avoids one RTEMS task
completely dominating the others.


## 11. Current Result

With the simpler independent-MO driver and the corrected latency-test
topology, the test successfully reached:

    high-priority sent:     1000
    high-priority received: 1000
    low-priority sent:      1000
    medium-priority sent:   1000

The lightweight ownership checks did not report duplicate TX Message
Object ownership during this test.

This gives me a stable baseline before adding more complicated scheduling
features.


## 12. Possible Future Research

I think the preemption experiments are still useful even though I do not
plan to put the current experimental implementation directly into the
upstream driver.

They show an interesting research problem:

> How can an RTOS CAN driver use multiple hardware TX buffers while
> preserving priority, low latency, and correct queue ownership?

A future priority-aware TX scheduler could study three levels:

1. RTEMS software queue priority
2. DCAN hardware Message Object scheduling
3. CAN identifier arbitration priority

Possible measurements include:

- high-priority frame latency
- average latency
- worst-case latency
- p95 and p99 latency
- TX throughput
- number of preemptions
- number of hardware aborts
- low-priority starvation
- CPU overhead
- queue occupancy
- frame loss or duplication

This may be useful for future research or a paper about priority-aware CAN
scheduling on RTEMS.


## Conclusion

The TX priority experiments started as an attempt to improve the DCAN
multi-buffer implementation.

The experiments included:

- multiple TX Message Objects
- software queue priority
- CAN-ID priority
- hardware TX abort
- priority preemption
- Message Object reordering
- TX-window compaction
- queue-slot ownership tracing
- latency testing

The most important lesson was that TX priority is not only a sorting
problem.

It is also an ownership and synchronization problem between:

    RTEMS software queues
            +
    DCAN hardware Message Objects
            +
    asynchronous interrupts
            +
    CAN bus arbitration

For the upstream driver, I currently prefer the simpler independent
multi-TX-MO design.

The more advanced priority-preemption design can remain as future
research work and may become part of a later paper.