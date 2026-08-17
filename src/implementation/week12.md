# GSoC 2026 RTEMS DCAN Driver Development — Week 12

## Week 12 — CAN-ID Based TX Priority Window

### New Design Goal

After the first TX preemption implementation in Week 11, my mentor clarified that TX priority should follow the **CAN arbitration identifier**:

> **A lower CAN ID has higher transmission priority.**

Therefore, the four DCAN TX Message Objects should no longer be treated simply as independent TX buffers.

Instead, they should be treated as a **fixed-size ordered hardware TX window**.

The important invariant is:

```text
CAN_ID(MO9)
    <=
CAN_ID(MO10)
    <=
CAN_ID(MO11)
    <=
CAN_ID(MO12)
```

For example:

```text
MO9  = 54
MO10 = 55
MO11 = 56
MO12 = 57
```

If a new frame with CAN ID `49` arrives:

```text
49 < 54 < 55 < 56 < 57
```

the best four frames should remain in hardware:

```text
MO9  = 49
MO10 = 54
MO11 = 55
MO12 = 56
```

while the lowest-priority frame should return to the software queue:

```text
57 -> RTEMS TX queue
```

This is different from the Week 11 design, which simply selected and replaced one victim Message Object.

The new design maintains a **CAN-ID ordered TX hardware window**.

---

## Finding the Insertion Position

I added a helper function that extracts the standard CAN identifier from a frame:

```c
static uint32_t dcan_tx_frame_id(
  const struct can_frame *frame
)
{
  return frame->header.can_id & 0x7ffu;
}
```

The driver then scans the active TX window and determines where the new frame belongs.

For example:

```text
Hardware:
54 55 57 60

New:
56
```

The comparison becomes:

```text
56 < 54 ? no
56 < 55 ? no
56 < 57 ? yes
```

Therefore:

```text
first_index = 2
```

which corresponds to `MO11`.

Only the affected suffix needs to be reorganized:

```text
MO9  = 54    unchanged
MO10 = 55    unchanged

MO11 and MO12 must be reordered
```

Using a strict `<` comparison also keeps frames with equal CAN IDs behind frames that are already queued.

This helps preserve **FIFO behavior for frames with the same CAN identifier**.

---

## Two Reorder Operations

I introduced two types of TX window reorganization:

```c
enum dcan_tx_reorder_type {
  DCAN_TX_REORDER_NONE = 0,
  DCAN_TX_REORDER_PREEMPT,
  DCAN_TX_REORDER_COMPACT
};
```

They solve two different problems.

### 1. PREEMPT

`DCAN_TX_REORDER_PREEMPT` occurs when a new higher-priority CAN frame needs to enter a **full hardware TX window**.

For example:

```text
Hardware:
54 55 57 60

New:
56
```

The desired result is:

```text
Hardware:
54 55 56 57

Software queue:
60
```

The new frame is inserted into the correct position, the affected frames are shifted, and the lowest-priority frame is returned to the RTEMS TX queue.

### 2. COMPACT

`DCAN_TX_REORDER_COMPACT` occurs when an earlier Message Object finishes transmission and leaves a hole in the hardware window.

For example:

```text
MO9  = free
MO10 = 54
MO11 = 55
MO12 = 56

Software queue:
57
```

If the driver simply filled MO9 with ID `57`, the result would be:

```text
57 54 55 56
```

which breaks the hardware priority order.

Instead, the driver first compacts the existing frames:

```text
54 -> MO9
55 -> MO10
56 -> MO11
```

leaving:

```text
MO9  = 54
MO10 = 55
MO11 = 56
MO12 = free
```

The next queued frame can then safely fill the tail:

```text
57 -> MO12
```

The final hardware window becomes:

```text
54 55 56 57
```

This keeps the hardware window ordered after TX completion.

---

## Reorder State

A reorder operation cannot be completed in a single instruction.

The driver must perform several steps:

```text
Detect reorder
      |
      v
Request hardware abort
      |
      v
Wait until TxRqst becomes 0
      |
      v
Rewrite affected Message Objects
```

Therefore, I added state to remember the active operation:

```c
struct dcan_tx_reorder {
  bool active;
  enum dcan_tx_reorder_type type;
  unsigned int first_index;
  struct rtems_can_queue_slot *new_slot;
  struct rtems_can_queue_edge *new_edge;
};
```

This structure records:

- whether a reorder operation is active,
- whether the operation is `PREEMPT` or `COMPACT`,
- where the affected Message Object range begins,
- the new queued frame waiting to enter the hardware window.

This allows the operation to continue safely across multiple TX worker iterations.

---

## Starting Priority Preemption

When all TX Message Objects are occupied, the driver checks the next frame waiting in the RTEMS software TX queue.

Conceptually, the logic is:

```text
Is another reorder active?
        |
     yes -> wait
        |
       no
        v
Are all four hardware TX buffers occupied?
        |
     no -> normal filling
        |
       yes
        v
Get next frame from RTEMS TX queue
        |
        v
Find insertion index by CAN ID
```

If the new frame does not belong in the active hardware window:

```text
first_index == 4
```

it is simply returned to the software queue.

For example:

```text
Hardware:
49 54 55 56

New:
57
```

ID `57` has lower priority than every frame already in hardware.

Therefore:

```text
first_index = 4
```

and no preemption is required.

---

## Aborting Only the Affected Suffix

The new design does not always abort all four Message Objects.

For example:

```text
Hardware:
54 56 57 60

New:
55
```

The insertion point is:

```text
first_index = 1
```

Therefore:

```text
MO9 stays unchanged

MO10
MO11
MO12
```

are the only affected Message Objects.

Similarly:

```text
Hardware:
54 55 57 60

New:
56
```

produces:

```text
first_index = 2
```

so only:

```text
MO11
MO12
```

need to be stopped and rewritten.

This avoids unnecessary abort and rewrite operations on Message Objects that are already in the correct position.

---

## Waiting for Hardware Abort Completion

The driver does not assume that calling the abort function means the Message Object is immediately safe to rewrite.

For every affected Message Object, the driver checks:

```text
TxRqst == 0
```

Only after all requested aborts have completed does the driver mark the reorder operation as ready.

The debug flow looks like:

```text
PREEMPT CHECK new_id=56 first_index=2
PREEMPT START first_index=2
REORDER READY type=1 first_index=2
```

This prevents the driver from rewriting a Message Object while the DCAN controller still considers its transmission pending.

---

## Completing the Ordered Insertion

Consider the following test:

```text
MO9  = 54
MO10 = 55
MO11 = 57
MO12 = 60

New = 56
```

The driver first saves the frame that will leave the hardware window:

```text
evicted = ID60
```

It then shifts the affected frames from right to left:

```text
MO12 <- old MO11 = 57
MO11 <- new frame = 56
```

The final hardware window becomes:

```text
54 55 56 57
```

while:

```text
60 -> software queue
```

The shift is performed **from right to left**.

This is important because shifting from left to right could overwrite information that is still needed for the next move.

---

## Preserving Physical Message Object Identity

Another important implementation detail is that I move only the software ownership information associated with a frame:

```text
slot
edge
priority
```

I do not copy the complete `dcan_txb_info` structure.

The reason is that `mobj` represents a physical DCAN hardware position:

```text
txb_info[0].mobj = 9
txb_info[1].mobj = 10
txb_info[2].mobj = 11
txb_info[3].mobj = 12
```

These hardware Message Object numbers must remain fixed.

Therefore, the driver moves the frame ownership information between TX buffer records while preserving the physical `mobj` identity of each buffer.

---

## TX Window Compaction

After a frame finishes transmission, the TX completion handler releases its RTEMS queue slot.

For example:

```text
Before:
49 54 55 56
```

After ID `49` finishes:

```text
free 54 55 56
```

Before performing a normal queue refill, the driver checks whether there is a hole before another active TX frame.

If there is, it starts:

```text
DCAN_TX_REORDER_COMPACT
```

The remaining frames are moved toward the lower-numbered Message Objects:

```text
free 54 55 56
       |
       v
54 55 56 free
```

The next software frame can then safely fill the tail:

```text
54 55 56 57
```

This solves the ordering problem observed during the earlier multi-buffer TX tests.

---

## Updated TX Processing Order

The TX worker now conceptually processes TX work in the following order:

```text
1. Finish an active abort/reorder
2. Compact a hardware window that contains a hole
3. Fill normal free TX Message Objects
4. Check whether a new queued frame should preempt the full window
```

This ordering is important.

If normal filling occurred before compaction, a lower-priority frame could enter an earlier Message Object and break the CAN-ID ordering again.

The driver therefore restores the hardware-window invariant before accepting additional frames.

---

## Week 12 Testing

I changed the local test so that it tests actual **CAN IDs** instead of only RTEMS queue priorities.

All test software queues use the same RTEMS queue priority.

This isolates:

```text
CAN-ID scheduling
```

from:

```text
RTEMS software queue priority
```

The payload also contains a sequence number so that I can independently verify:

```text
CAN ID   -> priority behavior
sequence -> frame identity, loss, or duplication
```

---

## Test 1 — Front Insertion

Initial hardware:

```text
54 55 56 57
```

New frame:

```text
49
```

Expected result:

```text
49 54 55 56

57 -> software queue
```

The driver reported:

```text
PREEMPT CHECK new_id=49 first_index=0
PREEMPT START first_index=0
REORDER READY type=1 first_index=0

TX REORDER: MO9=49 MO10=54 MO11=55 MO12=56
TX REQUEUE ID=57
```

Final receive order:

```text
49 54 55 56 57
```

**Result: PASS**

This test verified insertion at the front of the hardware TX window.

---

## Test 2 — Middle Insertion

Initial hardware:

```text
54 56 57 60
```

New frame:

```text
55
```

Expected result:

```text
54 55 56 57

60 -> software queue
```

The driver reported:

```text
PREEMPT CHECK new_id=55 first_index=1
PREEMPT START first_index=1
REORDER READY type=1 first_index=1

TX REORDER: MO9=54 MO10=55 MO11=56 MO12=57
TX REQUEUE ID=60
```

Final receive order:

```text
54 55 56 57 60
```

**Result: PASS**

This test proved that the driver can perform a **partial-range reorder** while keeping MO9 unchanged.

---

## Test 3 — Tail-Range Insertion

Initial hardware:

```text
54 55 57 60
```

New frame:

```text
56
```

Expected result:

```text
54 55 56 57

60 -> software queue
```

The driver reported:

```text
PREEMPT CHECK new_id=56 first_index=2
PREEMPT START first_index=2
REORDER READY type=1 first_index=2

TX REORDER: MO9=54 MO10=55 MO11=56 MO12=57
TX REQUEUE ID=60
```

Final receive order:

```text
54 55 56 57 60
```

Sequence order:

```text
100 101 1 102 103
```

**Result: PASS**

The test also reported:

```text
unique original frames:       4/4
every expected sequence once: yes
invalid frames:               0
error frames:                 0
```

This confirmed that the reorder operation did not lose or duplicate any frame.

---

## What I Learned

The work from Weeks 10–12 helped me understand that TX scheduling is not only about putting frames into hardware.

Several layers must remain consistent:

```text
RTEMS software TX queue
          |
          v
Queue slot / edge ownership
          |
          v
DCAN Message Object assignment
          |
          v
TxRqst hardware state
          |
          v
CAN arbitration priority
```

The biggest change in my understanding was moving from:

> **"Find one low-priority victim."**

to:

> **"Maintain an ordered hardware TX priority window."**

This model makes the driver behavior easier to reason about.

The main invariant is now:

> **Occupied TX Message Objects should remain ordered from lower CAN ID to higher CAN ID.**

When a higher-priority frame arrives, the driver performs an **ordered insertion**.

When a frame finishes and leaves a hole, the driver performs **compaction**.

Together, these two operations keep the active hardware TX window consistent with the intended CAN-ID priority model.

---

## Current Status

At the end of Week 12, the DCAN driver supports and has locally tested:

- Multiple TX Message Objects
- TX completion handling
- TX abort using the correct DCAN `TxRqst` semantics
- CAN-ID based priority comparison
- Priority insertion into a full TX hardware window
- Partial-range Message Object reordering
- Evicted frame return to the RTEMS software queue
- Post-completion TX window compaction
- Front insertion (`first_index = 0`)
- Middle insertion (`first_index = 1`)
- Tail-range insertion (`first_index = 2`)
- Frame-loss and duplicate-frame checking

The main tested example now behaves as expected:

```text
Initial:

MO9  = 54
MO10 = 55
MO11 = 56
MO12 = 57

New high-priority frame:

49

After preemption:

MO9  = 49
MO10 = 54
MO11 = 55
MO12 = 56

Software queue:

57
```

This implementation is much closer to the intended DCAN TX priority scheduling behavior and addresses the main mentor feedback about TX priority preemption.

---

## Next Steps

The next step is to continue testing edge cases before cleaning the code for the merge request.

An important remaining case is:

### `first_index = 3`

```text
Initial:
54 55 56 60

New:
57
```

Expected result:

```text
54 55 56 57

60 -> software queue
```

I also want to test:

- Equal CAN IDs and FIFO behavior
- A new frame with lower priority than every hardware frame
- Repeated preemptions
- Preemption while TX completion interrupts are occurring
- Heavier CAN bus load
- Cleanup of temporary debug output
- Final review of comments and error paths before upstream submission

The driver is now moving from basic functional TX support toward more robust scheduling behavior suitable for upstream review.