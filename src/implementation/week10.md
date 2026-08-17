# GSoC 2026 RTEMS DCAN Driver Development — Week 10

## Week 10 — Multi-Buffer TX and FIFO Behavior

### Background

The AM335x DCAN controller provides multiple Message Objects. In my driver, I use **Message Objects 9 through 12** as TX buffers:

- MO9
- MO10
- MO11
- MO12

Using several TX Message Objects improves throughput because the driver can prepare multiple CAN frames in hardware instead of waiting for one frame to finish before submitting the next one.

The basic TX flow is:

```text
RTEMS TX queue
      |
      v
Find a free DCAN TX Message Object
      |
      v
Write CAN frame into the Message Object
      |
      v
Set TxRqst
      |
      v
DCAN transmits the frame
      |
      v
TX completion interrupt
      |
      v
Release the RTEMS queue slot
```

At first, the driver could successfully transmit frames using several Message Objects, but I needed to verify that the transmission order was still correct.

---

## FIFO Problem

Suppose four frames are submitted in this order:

```text
Frame A
Frame B
Frame C
Frame D
```

They may be loaded into the hardware TX buffers as:

```text
MO9  = A
MO10 = B
MO11 = C
MO12 = D
```

A simple expectation is:

```text
A -> B -> C -> D
```

However, using multiple hardware Message Objects makes the situation more complicated.

The software queue may be FIFO, but the hardware Message Object positions can also affect which pending transmission is selected.

During testing, I observed results where frames could be received in a different order, for example:

```text
100 101 103 102
```

instead of:

```text
100 101 102 103
```

This showed that simply filling every free Message Object was not enough to guarantee the desired ordering.

---

## Understanding the Cause

The important lesson was that there are two different layers involved in TX ordering:

```text
Software TX queue ordering
          +
DCAN hardware Message Object ordering
```

The software queue may return frames in the correct order, but once several frames are already pending inside DCAN, the hardware decides which pending Message Object is transmitted next.

Therefore, the driver has to carefully manage the relationship between:

- RTEMS queue slot
- RTEMS queue edge
- DCAN Message Object
- `TxRqst` state

The driver cannot only consider the software queue. It also has to account for the state of the hardware TX buffers.

---

## TX Completion Tracking

For each hardware TX buffer, I keep information such as:

```c
struct dcan_txb_info {
  struct rtems_can_queue_slot *slot;
  struct rtems_can_queue_edge *edge;
  unsigned int mobj;
  int prio;
  bool abort_requested;
};
```

This structure connects each DCAN Message Object with the RTEMS frame currently using it.

When a transmission finishes, the driver:

1. Identifies the Message Object that generated the interrupt.
2. Checks the TX result.
3. Releases the corresponding RTEMS queue slot.
4. Clears the software ownership of the Message Object.
5. Allows another queued frame to use the free TX buffer.

The TX buffer state can therefore be viewed as:

```text
RTEMS queue
     |
     v
Queue slot + queue edge
     |
     v
dcan_txb_info
     |
     v
DCAN Message Object
     |
     v
Hardware transmission
     |
     v
TX completion interrupt
     |
     v
Release queue ownership
```

Keeping this relationship explicit became increasingly important as the TX implementation became more complex.

It also created the foundation for implementing **priority scheduling and TX preemption** in the following weeks.

---

## Week 10 Result

By the end of Week 10, the driver could use multiple TX Message Objects and correctly track their queue ownership and TX completion.

The main lesson from this week was:

> **A software FIFO alone does not automatically guarantee transmission order when several hardware TX buffers are active.**

The driver must consider both the ordering of the RTEMS software queues and the state of the DCAN hardware Message Objects.

This became particularly important when I started implementing **priority scheduling and runtime TX preemption** in the following stage of the project.