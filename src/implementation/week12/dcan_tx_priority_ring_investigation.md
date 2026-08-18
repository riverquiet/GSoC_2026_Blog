# DCAN TX Priority Ring Investigation and Validation

## 1. Background

TX scheduling was one of the most difficult parts of the DCAN driver
project.

The RTEMS CAN stack can provide multiple software TX queues with
different priority classes. The driver retrieves the oldest frame from
the highest-priority active queue by calling:

```c 
rtems_can_queue_test_outslot(qends, &qedge, &slot); 
```

The returned `qedge->edge_prio` tells the driver which RTEMS CAN
queue priority class the frame belongs to.

For the current experiment, three RTEMS CAN priority classes are used:

```text 
priority 2: high priority 1: middle priority 0: low 
```

The DCAN TX Message Objects are divided into three fixed groups:

```text 
RTEMS priority 2 -> MO9 / MO10 RTEMS priority 1 -> MO11 /
MO12 RTEMS priority 0 -> MO15 / MO16 / MO17 / MO18
```

Each priority class is managed as an independent TX ring. The current
experimental configuration does not require all rings to have the same
size: the high-priority ring remains small, while the low-priority ring
has been expanded to four Message Objects.

## 2. Why the Original Multi-MO Approach Was Difficult

The first multi-MO implementation treated several TX Message Objects as
independent free hardware buffers.

For example:

```text software queue: 0 1 2 3

hardware: MO9 = 0 MO10 = 1 MO11 = 2 MO12 = 3 
```

All four Message Objects could have `TxRqst` set at the same time.
After the frames were placed into independent hardware Message Objects,
the original software FIFO order was no longer automatically represented
by the hardware.

Testing showed receive patterns such as:

```text 0 2 1 3 4 6 5 7 8 10 9 11 
```

Therefore, simply filling multiple Message Objects was not enough to
preserve RTEMS queue order.

Another problem appeared when low-priority frames occupied all available
hardware TX Message Objects. If a high-priority frame arrived later, the
driver would need to decide whether to abort a low-priority MO, move a
frame to another MO, push a queue slot back, or rearrange the hardware
window.

This also made RTEMS queue-slot ownership difficult to manage correctly.

## 3. Queue Slot Ownership

The current driver keeps both the RTEMS queue slot and its queue edge in
each hardware TX buffer record:

```c struct dcan_txb_info { struct rtems_can_queue_slot *slot;
struct rtems_can_queue_edge *edge; int prio; unsigned int mobj;

bool active; bool cached; bool done; }; 
```

The ownership model is:

```text RTEMS software queue owns slot | v dcan_process_tx()
retrieves slot | v DCAN Message Object owns slot | v hardware
transmission completes | v ring retires the frame | v
rtems_can_queue_free_outslot() | v slot is returned to RTEMS 
```

This is important because the slot is not free immediately after it is
copied into a hardware Message Object. The driver must keep the slot
until the hardware transmission has completed and the frame has been
retired in ring order.

The new design avoids moving the same slot between different Message
Objects.

## 4. Priority-Specific Hardware Rings

Instead of sharing one hardware TX pool among all priority classes, the
new design gives each RTEMS priority class its own fixed hardware
resources:

```text RTEMS TX queues

      p2          p1          p0
      \|           |           |
      v           v           v
   ring[2]     ring[1]     ring[0]
      \|           |           |
   MO9/10      MO11/12     MO13/14

```

This has two important advantages.

### 4.1 Same-Priority Ordering

Each priority class has a small ordered software view over its fixed
hardware Message Objects.

```c 
struct dcan_tx_ring { unsigned int first_index; unsigned int
obj_num; unsigned int head; unsigned int tail; }; 
```

The number of outstanding frames is:

```c 
used = head - tail; 
```

The physical Message Object position is calculated only when needed:

```c 
position = logical_position % obj_num; 
```

For a two-object ring:

```text 
logical position: 0 1 2 3 4 5 ... physical object: A B A B A B ... 
```

`head` is the next producer position.

`tail` is the oldest frame that has not yet been retired.

### 4.2 High-Priority Frames Have Reserved Hardware Resources

A low-priority backlog can fill MO13/MO14, but it cannot use MO9/MO10.

Therefore:

```text 
p0 backlog | +-> MO13 +-> MO14 +-> remaining frames stay
in RTEMS software queue

later p2 frame arrives | +-> MO9 or MO10 
```

The high-priority frame does not require the driver to abort, move, or
rearrange the low-priority Message Objects.

## 5. Ring Retirement

Hardware completion and software retirement are different events.

When a TX completion is detected, the driver marks the corresponding
Message Object:

```text 
active = false done = true 
```

The driver then calls the ring retirement logic.

The retirement function always starts from `tail`.

For example:

``` text 
FIFO order:

A -> MO13 B -> MO14

head = 2 tail = 0 
```

If completion of B is observed first:

```text 
A.done = false B.done = true 
```

the driver does not retire B first.

The current consumer is still:

```text 
tail -> A 
```

Since A is not complete, retirement stops.

After A is complete:

```text 
retire A tail = 1

retire B tail = 2 
```

This preserves software ownership order even if completion observation
is not perfectly ordered.

A frame is removed from ownership only when:

```text 
1. the ring is not empty; 
2. the Message Object at tail owns a valid slot; 
3. that frame is marked done; 
4. no other Message Object owns the same RTEMS queue slot. 
```

After these checks, the driver updates TX statistics, generates local
echo, clears Message Object ownership, calls
`rtems_can_queue_free_outslot()`, and increments `tail`.

## 6. Same-Priority FIFO Validation

A focused test used only RTEMS priority 2 and the MO9/MO10 ring.

The test transmitted 64 frames with sequence numbers:

```text 
200..263 
```

Both Message Objects were used:

```text 
MO9 -> 200, 202, 204, ... 
MO10 -> 201, 203, 205, ... 
```

The final result was:

```text 
Expected frames: 64 Received frames: 64 FIFO errors: 0
Invalid frames: 0 CAN error frames: 0 
```

This showed that the current two-object ring management can preserve
same-priority FIFO order under the tested workload.

## 7. Three-Priority Validation

The driver was then tested with three RTEMS CAN priority classes:

```text 
p2 -> MO9/MO10 p1 -> MO11/MO12 p0 -> MO13/MO14 
```

Three sender tasks used the same RTEMS task priority so that the RTEMS
task scheduler would not become the main experimental variable.

Only the CAN queue priorities were different.

The test sent 256 frames for each priority class.

The result was:

```text 
p2: received=256, FIFO errors=0 p1: received=256, FIFO
errors=0 p0: received=256, FIFO errors=0

invalid frames: 0 CAN error frames: 0 
```

This demonstrated that all three fixed priority rings could operate
together while preserving FIFO order inside each priority class.

## 8. High-Priority Bypass Validation

The most important targeted test asked:

> Can a high-priority frame arriving later bypass an existing
low-priority backlog without rearranging the low-priority Message
Objects?

DCAN0 was the sender and DCAN1 was the receiver.

DCAN1 remained stopped so that DCAN0 could not receive acknowledgements.

Sixteen low-priority frames were queued:

```text 
p0 CAN ID = 0x600 sequence = 100..115 
```

The driver loaded the first two low-priority frames into the dedicated
low-priority ring:

```text 
MO13 = p0 seq100 ACTIVE MO14 = p0 seq101 ACTIVE 
```

The remaining low-priority frames stayed in RTEMS software queue.

A high-priority frame was then queued later:

```text 
p2 CAN ID = 0x020 sequence = 1 
```

The driver immediately placed it into the high-priority ring:

```text 
MO9 = p2 seq1 ACTIVE 
```

The hardware state before restoring ACK was:

```text 
MO9 = HIGH seq1 MO13 = LOW seq100 MO14 = LOW seq101

software p0 backlog: 102..115 
```

No low-priority Message Object was aborted or rearranged.

After DCAN1 was started, the receive order started with:

```text 
HIGH seq1 LOW seq100 LOW seq101 LOW seq102 ... LOW seq115
```

The final result was:

```text 
expected total: 17 received total: 17 low received: 16/16
high received: 1/1 low FIFO errors: 0 high receive position: 0 low
frames before high: 0 high-priority bypass: PASS invalid frames: 0 CAN
error frames: 0 
```

This directly demonstrated that dedicated priority-specific Message
Object groups provide an independent hardware path for a later arriving
high-priority frame.

This should be described as **priority bypass**, not frame
preemption. CAN transmission itself is non-preemptive. A frame already
transmitting on the bus cannot be interrupted in the middle.

## 9. Why the Current Design Is Simpler

The old approach used a shared hardware TX pool:

```text 
MO9 MO10 MO11 MO12 
```

If low-priority traffic occupied the whole pool, inserting a
high-priority frame required complex decisions about aborting, moving,
and reassigning ownership.

The current design changes the problem:

```text 
p2 owns MO9/MO10 p1 owns MO11/MO12 p0 owns MO13/MO14 
```

Therefore, low-priority traffic cannot consume the hardware resources
reserved for higher-priority traffic.

The design avoids:

```text 
moving RTEMS queue slots between Message Objects aborting
low-priority frames only to create space rebuilding the complete
hardware TX window complex slot-ownership reassignment 
```

## 10. RTEMS CAN Queue Priority

RTEMS CAN queues, also called edges, can belong to different priority
classes.

The official RTEMS CAN driver documentation states that
`rtems_can_queue_test_outslot()` retrieves the oldest ready slot from
the highest-priority active queue.

This priority is the software CAN queue priority, not CAN identifier
arbitration priority and not RTEMS task priority.

For the current experiments:

```text 
2 = high 1 = middle 0 = low 
```

Therefore:

```text 
queue priority 2 | v qedge->edge_prio = 2 | v tx_ring[2]
| v MO9/MO10 
```

Do not confuse this with RTEMS task priority, where smaller numeric
values normally represent higher task priority.

## 11. Testing Without TX Ring Trace

During development:

```c 
#define DCAN_TX_RING_TRACE 1 
```

enables output such as:

```text 
DCAN TX SUBMIT ... DCAN TX DONE ... DCAN TX RETIRE ... 
```

The next validation step is:

```c 
#define DCAN_TX_RING_TRACE 0 
```

and then rebuild and repeat the tests.

The purpose is to verify that the driver still behaves correctly without
frequent `printf()` calls changing worker timing.

Recommended regression tests:

### Test 1: Same-Priority FIFO

```text 
priority: 2 ring: MO9/MO10 frames: 64 expected: FIFO errors = 0 
```

### Test 2: Three-Priority Stress

```text 
p2 -> MO9/MO10 p1 -> MO11/MO12 p0 -> MO13/MO14

expected: all frames received FIFO errors = 0 for all classes 
```

### Test 3: Low-Backlog / High-Arrival Bypass

```text 
p0 backlog first p2 arrives later

expected: high bypasses remaining p0 software backlog low FIFO remains
correct 
```

If these tests continue to pass with trace disabled, it provides
stronger evidence that the behavior comes from the scheduling design
itself rather than timing changes caused by debug printing.

## 12. Expanded Low-Priority Ring and Trace-Free Validation

After the two-object priority-ring design was working, the low-priority
ring was expanded from two Message Objects to four:
```
p2 high -> MO9 / MO10
p0 low  -> MO15 / MO16 / MO17 / MO18
```
The important design rule was kept unchanged:

Increasing the ring size does not increase the number of
simultaneously ACTIVE Message Objects inside one priority class.

Instead, the larger ring is used as a hardware-side prefetch/cache
window.

For the four-object p0 ring, the intended state before acknowledgements
are restored is:
```
MO15 = seq100 ACTIVE
MO16 = seq101 CACHED
MO17 = seq102 CACHED
MO18 = seq103 CACHED

RTEMS software queue:
seq104 .. seq115
```
Therefore, for a ring with obj_num = 4, there is still only one ACTIVE
transmission candidate from that priority class, while up to three
additional frames may already be copied into DCAN Message RAM as CACHED
frames.

The rotation is conceptually:
```
MO15: 100 ACTIVE ---- DONE
MO16: 101 CACHED ---- ACTIVE ---- DONE
MO17: 102 CACHED ---------------- ACTIVE ---- DONE
MO18: 103 CACHED ---------------------------- ACTIVE ---- DONE
MO15: 104 CACHED ---------------------------------------- ACTIVE ...
```

This keeps same-priority hardware arbitration from reordering several
ACTIVE frames while allowing the driver to pre-stage future frames.

### 12.1 Trace-Free Test

The validation was repeated with:
```
#define DCAN_TX_RING_TRACE 0
```
This was important because earlier experiments showed that frequent
debug printf() calls could change worker timing and hide ordering
problems.

The test configuration was:
```
sender:   DCAN0 /dev/can0
receiver: DCAN1 /dev/can1

LOW:
  RTEMS queue priority = 0
  CAN ID = 0x600
  ring = MO15-MO18
  sequence = 100..115

HIGH:
  RTEMS queue priority = 2
  CAN ID = 0x020
  ring = MO9/MO10
  sequence = 1
```
DCAN1 remained stopped while the low-priority backlog and the later
high-priority frame were prepared.

Before restoring acknowledgements, the expected state was:
```
MO9  = HIGH seq1 ACTIVE

MO15 = LOW seq100 ACTIVE
MO16 = LOW seq101 CACHED
MO17 = LOW seq102 CACHED
MO18 = LOW seq103 CACHED

remaining LOW software backlog:
seq104..115
```
After DCAN1 was started, the observed receive order was:
```
HIGH seq1

LOW seq100
LOW seq101
LOW seq102
LOW seq103
...
LOW seq115
```
The result was:
```
expected total:         17
received total:         17
low received:           16/16
high received:          1/1
low FIFO errors:        0
high receive position:  0
low frames before high: 0
high-priority bypass:   PASS
invalid frames:         0
CAN error frames:       0
```
This test is stronger than the earlier trace-enabled result because the
driver preserved both priority bypass and low-priority FIFO ordering
without TX-ring debug printing affecting the timing.

### 12.2 What the Larger Ring Means

The larger ring should not be interpreted as:

four p0 Message Objects competing to transmit simultaneously

The design is instead:
```
one ACTIVE
+
up to obj_num - 1 CACHED
```
For the current p0 ring:
```
obj_num = 4

1 ACTIVE
+
up to 3 CACHED
```
This separates two concerns:
```
ACTIVE  -> hardware transmission eligibility
CACHED  -> hardware-side prefetch / ownership storage
```
The main purpose of increasing ring size is therefore not to create more
same-priority hardware transmission candidates. It is to increase the
number of frames that can be prefetched from the RTEMS software queue
into DCAN Message RAM while still preserving explicit FIFO activation
order.

### 12.3 Potential Performance Benefit

A larger cache window may improve TX pipeline efficiency, but it does
not increase the configured CAN bitrate.

Without prefetching, after a completion the driver may need to perform
the complete path:
```
TX completion
      |
      v
worker processing
      |
      v
retrieve next RTEMS queue slot
      |
      v
copy frame through DCAN IF registers
      |
      v
load Message Object
      |
      v
set TxRqst
```
With cached Message Objects, the next frame has already been copied into
DCAN Message RAM:
```
TX completion
      |
      v
retire old tail
      |
      v
next tail is CACHED
      |
      v
activate cached Message Object
```
The possible benefit is therefore reduced dependence on immediate queue
retrieval and Message Object loading after every completion.

Potential metrics for a future benchmark include:
```
1. total time to transmit N frames
2. inter-frame gap
3. TX-completion-to-next-activation latency
4. worker scheduling sensitivity / jitter
5. CPU processing overhead
```
Useful configurations to compare are:
```
ring size = 1
ring size = 2
ring size = 4
ring size = 8
```
The expectation should not be stated as "larger is always faster." The
current one-ACTIVE policy still requires software to activate the next
cached frame after completion. The performance benefit therefore needs
to be measured experimentally.

A more precise description of the architecture is:

The priority ring maintains an ordered software view over a fixed set
of DCAN TX Message Objects. One Message Object is
transmission-eligible at a time for each priority class, while
additional occupied Message Objects can serve as prefetched cached
frames.

## 13. Long-Run Same-Priority FIFO Stress Validation

After the targeted FIFO and priority-bypass tests passed, a longer
stress test was added using the mentor test's core idea: transmit a
large sequence of frames, encode a sequence number into the payload, and
verify the receive order explicitly.

The test configuration was:

``` text
sender:              DCAN0 /dev/can0
receiver:            DCAN1 /dev/can1
CAN bitrate:         125000 bit/s
fixed CAN ID:        0x600
frames per test:     2000
expected sequence:   0..1999
burst size:          100
burst delay:         1 ms
TX ring trace:       disabled
```

The sequence number was stored in `data[0..3]`. The receiver
reconstructed the sequence and compared every frame with the expected
value. A fixed CAN ID was used deliberately so this experiment focused
on driver ordering rather than CAN-ID arbitration.

The long run repeatedly exercised `head`, `tail`, physical Message
Object wrap-around, `ACTIVE -> DONE -> RETIRE`, `CACHED -> ACTIVE`, and
refill from the RTEMS software queue.

### 13.1 Priority 0: MO15-MO18

``` text
RTEMS queue priority: 0
TX ring:              MO15-MO18
ring size:            4
sent frames:          2000
received frames:      2000
order errors:         0
invalid frames:       0
CAN error frames:     0
timeout:              no
verified sequence:    0..1999
elapsed time:         1899282708 ns
```

Result: **PASS**

### 13.2 Priority 1: MO11-MO14

``` text
RTEMS queue priority: 1
TX ring:              MO11-MO14
ring size:            4
sent frames:          2000
received frames:      2000
order errors:         0
invalid frames:       0
CAN error frames:     0
timeout:              no
verified sequence:    0..1999
elapsed time:         1899274708 ns
```

Result: **PASS**

### 13.3 Priority 2: MO9-MO10

``` text
RTEMS queue priority: 2
TX ring:              MO9-MO10
ring size:            2
sent frames:          2000
received frames:      2000
order errors:         0
invalid frames:       0
CAN error frames:     0
timeout:              no
verified sequence:    0..1999
elapsed time:         1899186625 ns
```

Result: **PASS**

### 13.4 Combined Result

  ---------------------------------------------------------------------------
  RTEMS      TX Message      Ring size        Frames  Order errors Result
  priority   Objects                                               
  ---------- ----------- ------------- ------------- ------------- ----------
  p2         MO9-MO10                2          2000             0 PASS

  p1         MO11-MO14               4          2000             0 PASS

  p0         MO15-MO18               4          2000             0 PASS
  ---------------------------------------------------------------------------

The current variable-size priority-ring implementation therefore
preserved same-priority FIFO ordering for all three RTEMS CAN queue
priority classes under the tested long-run workload.

This supports the current invariant:

``` text
Within one priority ring:

at most one Message Object is ACTIVE
+
remaining occupied Message Objects may be CACHED
+
activation follows logical FIFO order
+
retirement follows tail order
```

## 14. Interpretation of the Timing Results

The total times were nearly identical:

  Priority     Ring size    Elapsed time
  ---------- ----------- ---------------
  p0                   4   1.899282708 s
  p1                   4   1.899274708 s
  p2                   2   1.899186625 s

These measurements do not show a meaningful throughput improvement from
ring size 4 over ring size 2. That is consistent with the architecture
because increasing ring size does not create more same-priority
transmission candidates.

``` text
ring size 2:
1 ACTIVE + up to 1 CACHED

ring size 4:
1 ACTIVE + up to 3 CACHED
```

The larger ring increases hardware-side prefetch/cache depth. Any
performance benefit is therefore more likely to appear in
completion-to-next-activation latency, inter-frame gap, worker
scheduling sensitivity, CPU overhead, or burst absorption. The
2000-frame experiment should primarily be treated as a correctness and
stability test.

## 15. Current Validated TX Architecture

``` text
RTEMS CAN TX queues
        |
        +-- priority 2 (high)
        |       -> ring[2] -> MO9-MO10
        |
        +-- priority 1 (middle)
        |       -> ring[1] -> MO11-MO14
        |
        +-- priority 0 (low)
                -> ring[0] -> MO15-MO18
```

The scheduling path is:

``` text
RTEMS queue priority
        |
        v
select priority ring
        |
        v
prefetch frames into that ring
        |
        v
only the oldest eligible frame becomes ACTIVE
        |
        v
later frames remain CACHED
        |
        v
TX completion
        |
        v
tail-ordered retirement
        |
        v
activate next cached frame
```

The architecture has now been validated in two complementary ways:

1.  **Within-priority ordering:** p0, p1, and p2 each transmitted 2000
    frames with zero ordering errors.
2.  **Cross-priority bypass:** a later high-priority frame used its
    reserved hardware ring and bypassed a low-priority backlog while the
    low-priority stream remained FIFO-correct.

The second property should be described as **priority bypass**, not
frame preemption. CAN transmission itself remains non-preemptive.

## 16. Why the Current Design Solves the Earlier Problems

The earlier multi-MO design exposed several same-priority Message
Objects as independent hardware transmission candidates. Once frames
were distributed among them, software FIFO order was no longer
sufficient to determine hardware transmission order. Observed patterns
such as `0 2 1 3` demonstrated this problem.

The current design instead preserves an explicit logical order:

``` text
head = next producer position
tail = oldest frame not yet retired
```

It also separates Message Object state:

``` text
ACTIVE = eligible for hardware transmission
CACHED = frame and RTEMS ownership are staged, but TxRqst is not yet eligible
```

Therefore, expanding a ring from two to four Message Objects increases
staged frames without increasing same-priority hardware candidates.

Resource partitioning also simplifies priority handling. Low-priority
traffic cannot consume the Message Objects reserved for p1 or p2, so a
later high-priority frame does not require moving, aborting, or
reassigning low-priority RTEMS queue slots.

## 17. Current Conclusions

### What did not work well

-   Treating multiple DCAN TX Message Objects as unrelated free buffers
    did not automatically preserve RTEMS software FIFO order.
-   A shared hardware TX pool complicated later high-priority insertion
    because lower-priority traffic could occupy the available hardware
    resources.

### What worked better

The successful experimental architecture combines:

-   fixed Message Object groups for each RTEMS software priority;
-   an independent TX ring for each priority;
-   explicit RTEMS queue-slot ownership;
-   logical `head`/`tail` producer-consumer management;
-   one `ACTIVE` Message Object per priority ring;
-   additional occupied Message Objects as `CACHED` frames;
-   tail-ordered retirement;
-   reserved hardware resources for higher-priority traffic.

The current variable-size mapping is:

``` text
p2 high   -> MO9-MO10    ring size 2
p1 middle -> MO11-MO14   ring size 4
p0 low    -> MO15-MO18   ring size 4
```

The current experiments have demonstrated:

-   same-priority FIFO preservation for p0, p1, and p2;
-   2000/2000 frames received for each long-run priority test;
-   zero ordering errors in all three long-run tests;
-   repeated ring wrap and refill under the tested workload;
-   low-priority FIFO preservation with a later high-priority arrival;
-   high-priority bypass using reserved hardware resources;
-   no need to rearrange low-priority Message Objects in the tested
    bypass case;
-   correct behavior with `DCAN_TX_RING_TRACE` disabled.

A concise description of the architecture is:

> The driver maintains an ordered software view over fixed DCAN TX
> Message Objects for each RTEMS CAN queue priority. Only one Message
> Object per priority ring is transmission-eligible at a time, while
> additional occupied Message Objects may cache prefetched frames. This
> preserves same-priority FIFO order while reserving an independent
> hardware path for higher-priority traffic.

## 18. Remaining Work

1.  Repeat the 2000-frame p0, p1, and p2 tests multiple times to check
    repeatability.
2.  Run a simultaneous three-priority long-duration stress test using
    the new `2/4/4` ring sizes.
3.  Test a p0 backlog followed by a p1 arrival.
4.  Test a p1 backlog followed by a p2 arrival.
5.  Consider a changing-CAN-ID test to study interaction between driver
    priority and CAN bus arbitration.
6.  Benchmark ring sizes 1, 2, 4, and 8 using more sensitive metrics
    than total TX time.
7.  Measure completion-to-next-activation latency and inter-frame gap.
8.  Keep the one-`ACTIVE`-per-priority invariant while evaluating larger
    cache windows.
9.  Remove experimental trace and validation code before upstream
    submission.
10. Discuss with mentors whether per-priority Message Object ranges and
    ring sizes should remain fixed, become configurable, or be
    simplified for the first upstream version.

The current results provide strong functional evidence for the
priority-specific TX-ring architecture. They should still be described
as experimental validation rather than as a final upstream scheduling
policy.
