GSoC 2026 RTEMS DCAN Driver Development — Week 11

Week 11 — First Version of TX Priority Preemption

Why TX Preemption Is Needed

Originally, the driver only filled free TX Message Objects.

For example:

MO9  = low-priority frame
MO10 = low-priority frame
MO11 = low-priority frame
MO12 = low-priority frame

If a high-priority frame arrived while all four Message Objects were occupied, it had to remain in the RTEMS software queue.

That means:

Low-priority frames already in hardware
        |
        v
must all finish first
        |
        v
high-priority frame can finally enter hardware

This is not good for priority scheduling.

The mentor pointed me to the SJA1000 RTEMS driver. The SJA1000 driver checks whether the currently active hardware transmission has lower priority than a frame waiting in the software queue. If a higher-priority frame is waiting, it can abort the current TX request and return the old frame to the software queue.

The important idea is:

higher-priority frame waiting
        +
lower-priority frame still pending in hardware
        |
        v
abort the lower-priority transmission
        |
        v
return it to software queue
        |
        v
send the higher-priority frame first

Initial DCAN Preemption Design

Because DCAN has several TX Message Objects, my first implementation selected one lower-priority Message Object as the victim.

For example:

MO9  = low
MO10 = low
MO11 = low
MO12 = low

software queue:
high

The driver selected a victim, usually the lowest-priority or highest-numbered Message Object:

victim = MO12

Then:

abort MO12
return old MO12 frame to queue
put high-priority frame into MO12

This was the first working version of TX preemption.

Debugging DCAN TX Abort

The most important low-level bug during this work was related to the DCAN Interface Command Register.

At first, the abort function tried to clear TxRqst in the IF Message Control Register:

IF1MCTL.TxRqst = 0

but the command also included:

IF1CMD.TxRqst/NewDat = 1

The DCAN documentation explains that during a write operation, setting the TxRqst/NewDat command bit directly sets the Message Object TxRqst bit to one, independent of the value in the IF Message Control Register.

So my code was effectively doing:

clear TxRqst
        |
        v
command sets TxRqst again

The result was:

TXRQ before abort = 0x00000f00
TXRQ after abort  = 0x00000f00

The abort never completed.

The fix was to clear TxRqst through the Message Control field and write it back using the CONTROL command, without setting the command's TxRqst/NewDat bit.

After the fix:

ABORT BEFORE MO12 TXRQ=0x00000f00
ABORT AFTER  MO12 TXRQ=0x00000700

The bit for MO12 was successfully cleared.

Successful First Preemption Test

A test was created where four low-priority frames occupied MO9–MO12 and a higher-priority frame was queued later.

The driver successfully:

found the victim
-> aborted the victim
-> returned the old frame to the software queue
-> submitted the higher-priority frame

The test showed:

PREEMPT VICTIM MO12
PREEMPT ABORTED MO12
PREEMPT REQUEUE MO12

This proved that hardware TX abort and software requeue were working.

Limitation of the Week 11 Design

The first design still had an important problem.

Suppose the hardware contains CAN IDs:

MO9  = 54
MO10 = 55
MO11 = 56
MO12 = 57

and a new higher-priority CAN ID 49 arrives.

The Week 11 design could do:

MO9  = 54
MO10 = 55
MO11 = 56
MO12 = 49

57 -> queue

The high-priority frame entered hardware, but it was placed in MO12.

This did not fully match the CAN priority model.

The mentor clarified that the correct result should be:

MO9  = 49
MO10 = 54
MO11 = 55
MO12 = 56

57 -> queue

This led to the Week 12 redesign.