# GSoC 2026 RTEMS DCAN Driver Development — Week 11

## Week 11 — First Version of TX Priority Preemption

### Why TX Preemption Is Needed

Originally, the driver only filled free TX Message Objects.

For example:

```text
MO9  = low-priority frame
MO10 = low-priority frame
MO11 = low-priority frame
MO12 = low-priority frame
```

If a high-priority frame arrived while all four Message Objects were occupied, it had to remain in the RTEMS software queue.

That means:

```text
Low-priority frames already in hardware
              |
              v
     must all finish first
              |
              v
High-priority frame can finally enter hardware
```

This behavior limits the effectiveness of priority scheduling. Even if the RTEMS software queue correctly identifies a high-priority frame, the frame cannot be transmitted promptly if all hardware TX Message Objects are occupied by lower-priority traffic.

---

## Learning from the SJA1000 Driver

My mentor pointed me to the RTEMS SJA1000 CAN driver as a reference.

The SJA1000 driver checks whether the currently active hardware transmission has a lower priority than a frame waiting in the software queue.

If a higher-priority frame is waiting, the driver can abort the lower-priority TX request and return that frame to the software queue.

The basic idea is:

```text
Higher-priority frame waiting
              +
Lower-priority frame pending in hardware
              |
              v
Abort the lower-priority transmission
              |
              v
Return the old frame to the software queue
              |
              v
Send the higher-priority frame first
```

I used this idea as the starting point for implementing TX preemption in the DCAN driver.

However, DCAN is more complicated because the driver uses multiple hardware TX Message Objects instead of a single active TX buffer.

---

## Initial DCAN Preemption Design

Because DCAN uses several TX Message Objects, my first implementation selected one lower-priority Message Object as the **preemption victim**.

For example:

```text
Hardware:

MO9  = low
MO10 = low
MO11 = low
MO12 = low

Software queue:

high
```

When the higher-priority frame arrived, the driver selected a lower-priority Message Object as the victim.

For example:

```text
victim = MO12
```

The preemption sequence was:

```text
Abort MO12
    |
    v
Return the old MO12 frame to the software queue
    |
    v
Load the high-priority frame into MO12
    |
    v
Request transmission
```

This became the first working version of DCAN TX priority preemption.

---

## Debugging DCAN TX Abort

The most important low-level bug during this work was related to the DCAN **Interface Command Register**.

To preempt a frame, the driver first needs to cancel the pending transmission by clearing the Message Object's `TxRqst` state.

Initially, the abort function attempted to clear `TxRqst` in the Interface Message Control Register:

```text
IF1MCTL.TxRqst = 0
```

However, the Interface Command Register also included:

```text
IF1CMD.TxRqst/NewDat = 1
```

According to the DCAN hardware behavior, during a write operation the `TxRqst/NewDat` command bit can directly set the Message Object's `TxRqst` state.

Therefore, my original abort sequence was effectively doing:

```text
Clear TxRqst in IF1MCTL
          |
          v
Execute IF1 write command
          |
          v
TxRqst/NewDat command bit sets TxRqst again
```

As a result, the transmission request was never actually cleared.

The debugging output showed:

```text
TXRQ before abort = 0x00000f00
TXRQ after abort  = 0x00000f00
```

The TX request remained active.

---

## Fixing the Abort Operation

The fix was to clear `TxRqst` through the Message Control field and write the updated control state back to the Message Object using the **CONTROL** transfer command, without setting the command's `TxRqst/NewDat` bit.

After this change, the debugging output became:

```text
ABORT BEFORE MO12 TXRQ=0x00000f00
ABORT AFTER  MO12 TXRQ=0x00000700
```

Before the abort:

```text
0x00000f00
```

indicated that MO9 through MO12 had pending TX requests.

After aborting MO12:

```text
0x00000700
```

the MO12 TX request bit was cleared successfully.

This confirmed that the driver could now cancel a pending DCAN hardware transmission.

---

## Successful First Preemption Test

I then created a test where four low-priority frames occupied MO9 through MO12 and a higher-priority frame was queued afterward.

The expected behavior was:

```text
Four low-priority frames
        |
        v
MO9–MO12 occupied
        |
        v
Higher-priority frame arrives
        |
        v
Select lower-priority victim
        |
        v
Abort victim
        |
        v
Return victim frame to RTEMS queue
        |
        v
Submit higher-priority frame
```

The driver successfully performed the complete sequence:

```text
PREEMPT VICTIM MO12
PREEMPT ABORTED MO12
PREEMPT REQUEUE MO12
```

This was an important milestone because it demonstrated that two key mechanisms were working together:

1. **Hardware TX abort**
2. **Software queue requeue**

The lower-priority frame was not simply discarded when preempted. Instead, ownership was returned to the software queue so that the frame could be transmitted later.

---

## Limitation of the Week 11 Design

Although the first preemption mechanism worked, it still had an important limitation.

Suppose the hardware currently contains frames with the following CAN IDs:

```text
MO9  = 54
MO10 = 55
MO11 = 56
MO12 = 57
```

Then a new higher-priority frame with CAN ID `49` arrives.

The Week 11 implementation could produce:

```text
MO9  = 54
MO10 = 55
MO11 = 56
MO12 = 49

Software queue:
57
```

The higher-priority frame successfully entered hardware, so the basic preemption mechanism worked.

However, it was simply placed into the Message Object that had been selected as the victim:

```text
MO12 = 49
```

This did not provide the hardware layout expected for the intended priority behavior.

My mentor clarified that the desired result should instead be:

```text
MO9  = 49
MO10 = 54
MO11 = 55
MO12 = 56

Software queue:
57
```

Conceptually, the TX window should change from:

```text
Before:

MO9     MO10    MO11    MO12
54      55      56      57

                 +
              new 49
```

to:

```text
After:

MO9     MO10    MO11    MO12
49      54      55      56

Queue:
57
```

The new high-priority frame should enter at the front of the active hardware TX window, while the existing frames are shifted and the lowest-priority frame is returned to the software queue.

---

## Why This Matters

Week 11 showed that **preemption is more than simply aborting one Message Object and replacing its frame**.

There are two separate problems:

```text
1. Which frame should be removed from hardware?

2. Where should the new high-priority frame be placed?
```

The first implementation solved the first problem:

> A lower-priority pending transmission could be aborted and safely returned to the RTEMS software queue.

But it did not yet fully solve the second problem:

> The remaining hardware Message Objects needed to be reorganized so that the active TX window reflected the desired priority ordering.

This distinction became the main focus of the next stage of development.

---

## Week 11 Result

By the end of Week 11, the driver could:

- Detect when a higher-priority frame was waiting.
- Identify a lower-priority hardware TX Message Object as a preemption victim.
- Correctly abort a pending DCAN transmission.
- Clear the corresponding `TxRqst` state.
- Return the preempted frame to the RTEMS software queue.
- Load the higher-priority frame into the available hardware TX buffer.
- Preserve the preempted frame instead of losing it.

The most important low-level lesson was:

> **Clearing `TxRqst` in the Message Control field is not sufficient if the Interface Command itself sets `TxRqst/NewDat` again.**

The larger scheduling lesson was:

> **Successful TX preemption requires both software queue management and careful management of the active hardware Message Object window.**

The Week 11 implementation proved that DCAN TX abort and requeue were possible, but it also exposed the limitation of replacing only a single victim Message Object.

That limitation led directly to the **Week 12 redesign**, where I began reorganizing the entire active TX Message Object window to preserve the intended priority order.