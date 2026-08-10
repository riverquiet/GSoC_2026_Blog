GSoC 2026 RTEMS DCAN Driver Development — Week 12

Week 12 — CAN-ID Based TX Priority Window

New Design Goal

The mentor clarified that TX priority should follow the CAN arbitration identifier:

A lower CAN ID has higher transmission priority.

Therefore, the four DCAN TX Message Objects should be treated as a fixed-size ordered hardware window.

The important invariant is:

CAN_ID(MO9)
    <=
CAN_ID(MO10)
    <=
CAN_ID(MO11)
    <=
CAN_ID(MO12)

For example:

MO9  = 54
MO10 = 55
MO11 = 56
MO12 = 57

If ID 49 arrives:

49 < 54 < 55 < 56 < 57

The best four frames should stay in hardware:

MO9  = 49
MO10 = 54
MO11 = 55
MO12 = 56

and the lowest-priority frame should return to the software queue:

57 -> RTEMS TX queue

This is different from simply replacing one victim.

The driver now maintains a CAN-ID ordered TX hardware window.

Finding the Insertion Position

I added a helper that extracts the standard CAN identifier:

static uint32_t dcan_tx_frame_id(
  const struct can_frame *frame
)
{
  return frame->header.can_id & 0x7ffu;
}

Then the driver scans the active TX window and finds where the new frame belongs.

Conceptually:

hardware:
54 55 57 60

new:
56

Comparison:

56 < 54 ? no
56 < 55 ? no
56 < 57 ? yes

Therefore:

first_index = 2

which corresponds to MO11.

The driver only needs to reorder the affected suffix:

MO9  = 54    unchanged
MO10 = 55    unchanged

MO11 and MO12 must be reordered

Using strict < also keeps equal CAN IDs behind already queued frames, which helps preserve FIFO behavior for frames with the same identifier.

Two Reorder Operations

I introduced two types of TX window reorganization:

enum dcan_tx_reorder_type {
  DCAN_TX_REORDER_NONE = 0,
  DCAN_TX_REORDER_PREEMPT,
  DCAN_TX_REORDER_COMPACT
};

They solve two different problems.

1. PREEMPT

This happens when a new higher-priority CAN frame needs to enter a full hardware TX window.

Example:

54 55 57 60
+
56

Result:

54 55 56 57

60 -> queue

2. COMPACT

This happens after an earlier Message Object finishes transmission and leaves a hole.

Example:

MO9  = free
MO10 = 54
MO11 = 55
MO12 = 56

queue:
57

If the driver simply fills MO9 with ID57, it would create:

57 54 55 56

which breaks the hardware priority order.

Instead, the driver compacts the existing frames:

54 -> MO9
55 -> MO10
56 -> MO11

and then fills the tail:

57 -> MO12

Result:

54 55 56 57

This keeps the hardware window ordered after every TX completion.

Preemption State

The reorder process cannot happen in one instruction.

The driver must:

detect reorder
    |
    v
request hardware abort
    |
    v
wait until TxRqst becomes 0
    |
    v
rewrite Message Objects

Therefore, I added state to remember the active operation:

struct dcan_tx_reorder {
  bool active;
  enum dcan_tx_reorder_type type;
  unsigned int first_index;
  struct rtems_can_queue_slot *new_slot;
  struct rtems_can_queue_edge *new_edge;
};

This stores:

whether a reorder is active,

whether it is PREEMPT or COMPACT,

where the affected range starts,

the new queued frame that is waiting to enter hardware.

Starting Priority Preemption

When all TX Message Objects are occupied, the driver checks the next software TX frame.

The logic is approximately:

Is another reorder active?
    |
    yes -> wait

Are all four hardware TX buffers occupied?
    |
    no -> normal filling

Get next frame from RTEMS TX queue
    |
    v
Find insertion index by CAN ID

If the new frame does not belong in the hardware window:

first_index == 4

it is simply returned to the software queue.

For example:

hardware:
49 54 55 56

new:
57

ID57 has lower priority than every active hardware frame:

first_index = 4

so no preemption is required.

Aborting Only the Affected Suffix

The new design does not always abort all four Message Objects.

For:

54 56 57 60 + 55

the insertion point is:

first_index = 1

so:

MO9 stays unchanged
MO10-MO12 are affected

For:

54 55 57 60 + 56

the insertion point is:

first_index = 2

so only:

MO11-MO12

need to be stopped and rewritten.

This reduces unnecessary hardware operations.

Waiting for Hardware Abort Completion

The driver does not assume that calling the abort function means the Message Object is immediately safe to rewrite.

For every affected Message Object, it checks:

TxRqst == 0

Only after all requested aborts are complete does the driver mark the reorder as ready.

The debug flow looks like:

PREEMPT CHECK new_id=56 first_index=2
PREEMPT START first_index=2
REORDER READY type=1 first_index=2

This protects the driver from rewriting a Message Object while the controller still considers its transmission pending.

Completing the Ordered Insertion

Consider this test:

MO9  = 54
MO10 = 55
MO11 = 57
MO12 = 60

new = 56

The driver saves the frame that will leave the window:

evicted = ID60

Then it shifts the affected frames from right to left:

MO12 <- old MO11 = 57
MO11 <- new frame = 56

The final hardware window becomes:

54 55 56 57

and:

60 -> software queue

The shift is done from right to left because doing it from left to right could overwrite data that still needs to be moved.

Another important detail is that I move only the software ownership information:

slot
edge
priority

I do not copy the complete dcan_txb_info structure because mobj represents a physical hardware position:

txb_info[0].mobj = 9
txb_info[1].mobj = 10
txb_info[2].mobj = 11
txb_info[3].mobj = 12

These hardware numbers must never move.

TX Window Compaction

After a frame finishes, the TX completion handler releases its queue slot.

For example:

before:
49 54 55 56

49 finishes

after completion:
free 54 55 56

Before a normal queue refill, the driver checks whether there is a hole before another active TX frame.

If there is, it starts:

DCAN_TX_REORDER_COMPACT

The remaining frames are moved toward lower-numbered Message Objects:

free 54 55 56
    |
    v
54 55 56 free

Then the next software frame can safely fill the tail:

54 55 56 57

This solves the ordering problem I observed during earlier tests.

Updated TX Processing Order

The TX worker now conceptually processes TX work in this order:

1. Finish an active abort/reorder
2. Compact a hardware window that contains a hole
3. Fill normal free TX Message Objects
4. Check whether a new queued frame should preempt the full window

This order is important.

If normal filling happened before compaction, a low-priority frame could enter an earlier Message Object and break CAN-ID ordering again.

Week 12 Testing

I changed the local test so that it tests actual CAN IDs instead of only RTEMS queue priorities.

All test software queues use the same RTEMS queue priority. This isolates CAN-ID scheduling from RTEMS software queue priority.

The payload also contains a sequence number so I can independently check:

CAN ID -> priority behavior
sequence -> frame identity, loss, or duplication

Test 1 — Front Insertion

Initial hardware:

54 55 56 57

New frame:

49

Expected:

49 54 55 56

57 -> queue

Driver result:

PREEMPT CHECK new_id=49 first_index=0
PREEMPT START first_index=0
REORDER READY type=1 first_index=0

TX REORDER: MO9=49 MO10=54 MO11=55 MO12=56
TX REQUEUE ID=57

Final receive order:

49 54 55 56 57

Result:

PASS

Test 2 — Middle Insertion

Initial hardware:

54 56 57 60

New frame:

55

Expected:

54 55 56 57

60 -> queue

Driver result:

PREEMPT CHECK new_id=55 first_index=1
PREEMPT START first_index=1
REORDER READY type=1 first_index=1

TX REORDER: MO9=54 MO10=55 MO11=56 MO12=57
TX REQUEUE ID=60

Final receive order:

54 55 56 57 60

Result:

PASS

This test proved that the driver can perform a partial-range reorder and keep MO9 unchanged.

Test 3 — Tail-Range Insertion

Initial hardware:

54 55 57 60

New frame:

56

Expected:

54 55 56 57

60 -> queue

Driver result:

PREEMPT CHECK new_id=56 first_index=2
PREEMPT START first_index=2
REORDER READY type=1 first_index=2

TX REORDER: MO9=54 MO10=55 MO11=56 MO12=57
TX REQUEUE ID=60

Final receive order:

54 55 56 57 60

Sequence order:

100 101 1 102 103

Result:

PASS

The test also reported:

unique original frames:       4/4
every expected sequence once: yes
invalid frames:               0
error frames:                 0

This confirms that the reorder operation did not lose or duplicate any frame.

What I Learned

These three weeks helped me understand that TX scheduling is not only about putting frames into hardware.

There are several layers that must stay consistent:

RTEMS software TX queue
        |
        v
queue slot / edge ownership
        |
        v
DCAN Message Object assignment
        |
        v
TxRqst hardware state
        |
        v
CAN arbitration priority

The biggest change in my understanding was moving from:

"find one low-priority victim"

to:

"maintain an ordered hardware TX priority window"

This model makes the driver logic much easier to reason about.

The main invariant is now:

occupied TX Message Objects should remain ordered
from lower CAN ID to higher CAN ID

When a higher-priority frame arrives, the driver performs an ordered insertion.

When a frame finishes and leaves a hole, the driver performs compaction.

Together, these two operations keep the hardware window consistent with CAN arbitration priority.

Current Status

At the end of Week 12, the DCAN driver supports and has locally tested:

multiple TX Message Objects,

TX completion handling,

TX abort using the correct DCAN TxRqst semantics,

CAN-ID based priority comparison,

priority insertion into a full TX hardware window,

partial-range Message Object reordering,

evicted frame return to the RTEMS software queue,

post-completion TX window compaction,

front insertion (first_index = 0),

middle insertion (first_index = 1),

tail-range insertion (first_index = 2),

frame-loss and duplicate-frame checking.

The main tested example now behaves as expected:

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

This implementation is much closer to the intended DCAN TX priority scheduling behavior and addresses the main mentor feedback about TX priority preemption.

Next Steps

The next step is to continue testing edge cases before cleaning the code for the merge request.

Important remaining tests include:

first_index = 3

Initial:
54 55 56 60

New:
57

Expected:
54 55 56 57

60 -> queue

I also want to test:

equal CAN IDs and FIFO behavior,

a new frame that has lower priority than every hardware frame,

repeated preemptions,

preemption while TX completion interrupts are occurring,

heavier CAN bus load,

cleanup of temporary debug output,

final review of comments and error paths before upstream submission.

The driver is now moving from basic functional support toward more robust scheduling behavior suitable for upstream review.
