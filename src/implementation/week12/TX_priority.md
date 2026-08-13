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

The goal was to improve TX capability and also study how TX priority
should work when several CAN frames are already waiting inside the DCAN
hardware.

This became more complicated than I expected, but the experiments helped
me understand the relationship between the RTEMS CAN queue, DCAN Message
Objects, and CAN bus arbitration.

---

## 1. Multi-TX Message Objects

With four TX Message Objects, the basic idea was:

```text
RTEMS CAN TX queue
        |
        v
   DCAN worker
        |
        v
+-------+-------+-------+-------+
|  MO9  | MO10  | MO11  | MO12 |
+-------+-------+-------+-------+
```

Each active Message Object keeps one RTEMS queue slot:

```text
MO9  <-> slot A
MO10 <-> slot B
MO11 <-> slot C
MO12 <-> slot D
```

This gives an important ownership rule:

> One RTEMS CAN queue slot should be owned by only one TX Message Object
> at one time.

When a Message Object finishes transmission, the driver returns a local
echo, updates the TX statistics, and releases the related queue slot.

This ownership model became one of the most important parts of the TX
implementation.

---

## 2. Why TX Priority Is More Complicated

CAN already has hardware arbitration.

For normal CAN frames:

> A lower CAN identifier has higher arbitration priority.

For example:

```text
CAN ID 0x020 -> high CAN arbitration priority
CAN ID 0x500 -> medium CAN arbitration priority
CAN ID 0x700 -> low CAN arbitration priority
```

However, RTEMS also has software TX queues with their own priorities.

This means there are several scheduling levels:

```text
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
```

A higher-priority frame may enter the RTEMS software queue after several
lower-priority frames have already been copied into hardware Message
Objects.

This was the reason I started experimenting with TX preemption.

---

## 3. First Preemption Idea

Suppose all four TX Message Objects are busy:

```text
MO9  -> low-priority frame
MO10 -> low-priority frame
MO11 -> low-priority frame
MO12 -> low-priority frame
```

Then a new high-priority frame arrives.

The idea was:

```text
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
```

This idea was partly inspired by the SJA1000 driver, which can abort a
pending hardware transmission when a higher-priority software frame is
waiting.

The basic idea is reasonable, but DCAN makes the implementation more
complicated because several hardware Message Objects may be active at the
same time.

---

## 4. CAN-ID Based Hardware Reordering

I later tested a more complete design.

The idea was to keep the active DCAN TX Message Objects ordered by CAN ID.

For example:

```text
MO9  -> ID 54
MO10 -> ID 55
MO11 -> ID 57
MO12 -> ID 60
```

Now a new frame with ID 56 arrives.

Because ID 56 has higher CAN arbitration priority than ID 57 and ID 60,
the desired hardware order becomes:

```text
MO9  -> ID 54
MO10 -> ID 55
MO11 -> ID 56
MO12 -> ID 57
```

and ID 60 is returned to the software queue.

The driver therefore needs to:

1. Find the correct insertion position.
2. Cancel the affected hardware TX requests.
3. Wait for `TxRqst` to clear.
4. Move the software ownership information.
5. Insert the new frame.
6. Return the displaced frame to the RTEMS queue.
7. Rewrite the affected DCAN Message Objects.

I created several controlled tests for front, middle, and tail insertion.

These tests showed that the basic CAN-ID ordering idea could work in
small deterministic cases.

However, the implementation also introduced much more state movement
between RTEMS queue slots and hardware Message Objects.

---

## 5. TX Compaction

Another problem happens when Message Objects finish independently.

For example:

```text
MO9  -> free
MO10 -> ID 54
MO11 -> ID 55
MO12 -> ID 56
```

Now there is a hole in the hardware TX window:

```text
[ free ][ 54 ][ 55 ][ 56 ]
```

I experimented with a compaction operation:

```text
before:

[ NULL, 54, 55, 56 ]

after:

[ 54, 55, 56, NULL ]
```

Then the driver can take another frame from the RTEMS TX queue and put it
into the last free Message Object.

This made the TX scheduling design more complete, but also much more
complex.

A compaction operation must update both:

```text
software ownership state
+
hardware Message Object state
```

while interrupts and TX completion may happen asynchronously.

---

## 6. A Timing-Sensitive Bug

During these experiments, I found an interesting problem.

When the driver contained many `printf()` debug messages, the preemption
code sometimes appeared to work correctly.

After removing the debug prints, the driver became much faster, but some
failures appeared.

I observed errors such as:

```text
DUPLICATE FROM QUEUE

queue returned owned slot

SLOT HAS MULTIPLE OWNERS

FIFO self-loop risk
```

This was an important clue.

The `printf()` calls were not fixing the driver. They were changing the
timing of the system.

The debug output slowed down:

- the worker task
- TX completion processing
- queue operations
- hardware Message Object updates

Because of this, some timing-sensitive problems became harder to
reproduce.

This is similar to a Heisenbug: adding debug code changes the timing and
therefore changes the behavior of the bug.

This taught me that a driver should always be tested again after debug
output is removed.

---

## 7. Queue Slot Ownership Became the Main Debugging Question

At first, I thought the main problem was the preemption algorithm.

Later, I realized that the more important question was:

> Who owns each RTEMS CAN queue slot at each point in time?

The expected ownership lifecycle is:

```text
RTEMS queue owns slot
        |
        | rtems_can_queue_test_outslot()
        v
DCAN driver owns slot
        |
        | hardware transmission
        v
TX completion
        |
        | rtems_can_queue_free_outslot()
        v
RTEMS queue owns slot again
```

A slot should never be owned by two TX Message Objects at the same time.

It should also not be returned by the RTEMS queue while the driver still
uses it.

I added ownership checks and a small in-memory trace to record operations
such as:

```text
GET
MOVE
REQUEUE
FREE
```

This helped show that some failures were related to queue-slot lifetime,
not only to CAN-ID preemption.

The most important invariant became:

> One queue slot must have exactly one owner while it is assigned to
> hardware.

---

## 8. A/B Test

To understand the problem better, I disabled runtime priority preemption
but kept the multi-TX-MO implementation.

The duplicate ownership problem could still appear.

This was an important result because it showed:

> The problem was not caused only by the preemption algorithm.

Preemption and compaction made the TX path more complicated and made the
problem easier to reproduce, but the deeper issue involved multiple
outstanding RTEMS TX queue slots and their ownership lifetime.

This experiment also showed that there were two different questions:

```text
1. Is queue-slot ownership correct?

2. Is the externally visible TX order correct?
```

Both must be correct before a multi-TX-MO implementation is suitable for
upstream use.

---

## 9. Same-Priority Ordering With Multiple TX Message Objects

After removing the complex preemption logic, I returned to a simpler
multi-TX-MO implementation.

The worker simply took frames from the RTEMS CAN queue and assigned them
to free TX Message Objects:

```text
RTEMS TX queue
        |
        v
   DCAN worker
        |
        +--> MO9
        +--> MO10
        +--> MO11
        +--> MO12
```

There was no runtime preemption, no Message Object sorting, and no
compaction.

Each Message Object independently owned one queue slot until transmission
completion.

This design was much more stable than the earlier preemption version.

However, another important requirement remained:

> Frames from the same RTEMS priority class should preserve the software
> queue order.

To test this, I created a same-priority FIFO test.

All frames used:

```text
same RTEMS TX queue
same queue priority
same CAN ID
```

The only difference was the sequence number in the CAN payload:

```text
0 1 2 3 4 5 6 7 ...
```

With four hardware TX Message Objects active, the received order showed a
very regular reordering pattern:

```text
0 2 1 3
4 6 5 7
8 10 9 11
12 14 13 15
...
```

This was not random packet loss.

All frames were still received, but their transmission order did not
always match the RTEMS software queue order.

This showed that:

> Simply filling several DCAN TX Message Objects from the RTEMS FIFO does
> not automatically preserve the externally visible FIFO transmission
> order.

The software queue may give frames to the driver in order, but once
several transmissions are simultaneously outstanding in different
hardware Message Objects, hardware scheduling becomes another part of
the ordering problem.

This result changed the final upstream design decision.

---

## 10. Returning to a Simpler Upstream Baseline

Because same-priority FIFO ordering is an important correctness property,
I decided to use a more conservative TX baseline for the current upstream
driver.

The current configuration uses only one hardware TX Message Object:

```text
RTEMS CAN TX queue
        |
        v
       MO9
        |
        v
     CAN bus
```

The current Message Object allocation is:

```text
RX:
MO1-MO8

TX:
MO9
```

This intentionally limits the number of outstanding hardware
transmissions to one.

The driver still keeps the generic TX ownership structure:

```text
txb_info[]
```

and the TX code is still written around `DCAN_TX_MO_COUNT`.

Therefore the architecture can be extended to multiple TX Message
Objects later, but the current configuration enables only one TX
Message Object.

This gives the current driver a simple property:

```text
RTEMS queue order
        |
        v
single hardware TX request
        |
        v
CAN transmission order
```

It avoids having several same-priority frames concurrently active in
different hardware TX Message Objects.

The TX and RX interface registers are also separated:

```text
IF1 -> RX and deferred Message Object / interrupt processing

IF2 -> TX
```

This separation was kept because it makes TX/RX register interaction
simpler and avoids unnecessary contention between the two paths.

---

## 11. Same-Priority FIFO Verification

I tested the single-TX-MO baseline with 64 frames.

All frames used:

```text
CAN ID         = 0x123
RTEMS priority = 0
sequence       = 0..63
```

The expected order was:

```text
0 1 2 3 4 5 ... 63
```

The result was:

```text
Expected frames:  64
Received frames:  64
Order errors:     0
Invalid frames:   0
CAN error frames: 0
Verified order:   0..63
```

The received sequence was exactly:

```text
0 1 2 3 4 5 6 7 8 9 ...
...
60 61 62 63
```

The test passed.

This confirms that the current single-TX-MO baseline preserves FIFO
ordering for frames from the same RTEMS priority class.

It also repeatedly exercises:

```text
RTEMS queue
    |
    v
test_outslot()
    |
    v
MO9 transmission
    |
    v
TX completion
    |
    v
local echo
    |
    v
free_outslot()
    |
    v
next frame
```

so the test verifies more than only the first hardware transmission.

---

## 12. Current Upstream TX Design

The current upstream design is intentionally simple.

The TX path does:

```text
RTEMS software TX queue
        |
        | rtems_can_queue_test_outslot()
        v
queue slot
        |
        v
bind slot / edge / priority
to TX ownership structure
        |
        v
write frame to MO9 using IF2
        |
        v
hardware transmission
        |
        v
TX completion interrupt
        |
        v
local echo + statistics
        |
        v
clear driver ownership
        |
        v
rtems_can_queue_free_outslot()
```

The current TX code still contains the future scheduling direction:

```c
/*
 * TODO: Call rtems_can_queue_pending_outslot_prio() and rearrange
 * TX message objects when a higher-priority class would be blocked
 * by a lower-priority one.
 */
```

This means the current upstream implementation focuses on correctness and
FIFO ordering first.

Advanced multi-TX-MO scheduling can be added later without changing the
basic queue-slot ownership model.

---

## 13. Current Latency Test Topology

During the TX experiments, I also created a latency test using both
AM335x DCAN controllers.

The topology was:

```text
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
```

All controllers were connected to the same physical CAN bus.

The receiver was mainly used to measure the high-priority ID `0x020`
traffic.

The test also used paced traffic instead of unlimited write loops.

For example:

```text
send 4 frames
     |
     v
wait 10 ms
     |
     v
send next 4 frames
```

This made the workload more balanced and prevented one RTEMS task from
completely dominating the others.

One long-running test successfully reached:

```text
high-priority sent:     1000
high-priority received: 1000
low-priority sent:      1000
medium-priority sent:   1000
```

This test was useful for checking long-running driver behavior and
ownership stability.

However, the later same-priority FIFO test showed that long-running
stability alone is not enough to prove software queue ordering when
multiple hardware TX Message Objects are active.

That distinction became important for the final upstream design.

---

## 14. What the Failed Experiments Taught Me

The preemption experiments did not become part of the final upstream
baseline, but they were still very useful.

They revealed several important problems.

### 14.1 Priority is not only sorting

At first, TX priority looked like a sorting problem:

```text
high priority first
low priority later
```

But in a real driver, the problem also includes:

```text
RTEMS software queues
        +
queue-slot ownership
        +
DCAN hardware Message Objects
        +
asynchronous TX completion
        +
interrupt processing
        +
worker scheduling
        +
hardware abort
        +
CAN arbitration
```

### 14.2 Hardware parallelism creates software complexity

Multiple hardware TX buffers can improve flexibility, but they also mean
that several software queue slots may be outstanding at the same time.

This makes ordering and ownership more difficult.

### 14.3 A successful small test is not enough

Several controlled preemption tests worked correctly.

However, larger tests exposed:

```text
duplicate frames
reordered frames
ownership problems
timing-sensitive behavior
```

This showed why stress testing is important before putting a scheduling
feature into an upstream driver.

### 14.4 Debug output can hide bugs

The behavior difference with and without `printf()` was a useful lesson.

A driver should not depend on debug output timing.

### 14.5 A smaller correct baseline is better than a larger unstable one

The final upstream baseline intentionally supports fewer simultaneous TX
Message Objects.

This is a tradeoff:

```text
less hardware parallelism
        |
        v
simpler ownership
        +
deterministic same-priority FIFO
        +
easier upstream review
```

For the current stage of the driver, this is the better engineering
choice.

---

## 15. Possible Future Work

The multi-TX-MO and preemption experiments still show an interesting
future development problem:

> How can an RTOS CAN driver use multiple hardware TX Message Objects while
> preserving software queue priority, FIFO ordering, and correct queue
> ownership?

A future implementation can investigate:

1. RTEMS software queue priority.
2. FIFO ordering within the same priority class.
3. DCAN hardware Message Object scheduling.
4. CAN identifier arbitration.
5. Runtime TX preemption.
6. Hardware abort and rescheduling.
7. Queue-slot ownership during asynchronous completion.

The planned upstream direction is to use:

```c
rtems_can_queue_pending_outslot_prio()
```

to determine whether a higher-priority software queue is waiting.

If a higher-priority class would otherwise be blocked by lower-priority
frames already assigned to hardware Message Objects, the driver could
rearrange or abort selected hardware TX requests.

However, a future implementation must preserve:

```text
same-priority FIFO
no frame loss
no frame duplication
correct queue-slot ownership
stable behavior without debug timing
```

Possible measurements include:

- high-priority frame latency
- average latency
- worst-case latency
- p95 latency
- p99 latency
- TX throughput
- number of preemptions
- number of hardware aborts
- low-priority starvation
- CPU overhead
- queue occupancy
- frame loss
- frame duplication
- ordering errors
- ownership errors

This may also be useful for future research on priority-aware CAN
scheduling in RTEMS.

---

## Conclusion

The TX priority work started as an attempt to improve the DCAN
multi-buffer implementation.

The experiments included:

- multiple TX Message Objects
- RTEMS software queue priority
- CAN-ID priority
- hardware TX abort
- priority preemption
- Message Object reordering
- TX-window compaction
- queue-slot ownership tracing
- latency testing
- same-priority FIFO testing

The most important lesson was that TX priority is not only a sorting
problem.

It is also an ownership and synchronization problem between:

```text
RTEMS software queues
        +
DCAN hardware Message Objects
        +
asynchronous interrupts
        +
worker scheduling
        +
CAN bus arbitration
```

The experiments also showed that simply using multiple independent TX
Message Objects does not automatically preserve the RTEMS software queue
order.

For the current upstream driver, I therefore chose a conservative
single-TX-MO baseline.

The current configuration uses:

```text
RX: MO1-MO8
TX: MO9
```

A 64-frame same-priority test verified:

```text
64 frames sent
64 frames received
0 ordering errors
0 invalid frames
0 CAN error frames
FIFO sequence 0..63 preserved
```

The code still keeps a structure that can support multiple TX Message
Objects in the future, but only one TX Message Object is currently
enabled.

This provides a simple and deterministic upstream baseline while the
more advanced multi-TX-MO priority-aware scheduler remains future work.

The preemption experiments were therefore still valuable even though the
experimental implementation is not included in the upstream driver.

They helped identify the scheduling, synchronization, ordering, and
queue-ownership problems that a future multi-TX-MO implementation will
need to solve.