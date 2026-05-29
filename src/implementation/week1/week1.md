# week_1

## Overall Conclusion
This week marked my first successful interaction with real hardware through RTEMS on the BeagleBone Black.

At the beginning of the project, my understanding of embedded systems was largely theoretical. I knew that RTEMS applications could be compiled into executable images, but I did not have a clear mental model of how source code actually became a running program on hardware.

Through building, deploying, and testing a minimal RTEMS application, I established a complete understanding of the development workflow:

```text
Source Code
    ↓
RTEMS Build
    ↓
rtems-app.img
    ↓
TFTP Transfer
    ↓
U-Boot
    ↓
BeagleBone Black
    ↓
ARM Cortex-A8 Execution
    ↓
Memory-Mapped Registers
    ↓
DCAN Peripheral
```

This realization significantly reduced the gap between software and hardware in my understanding of embedded systems.

Technically, the primary accomplishment of the week was successfully bringing up the AM335x D-CAN controller and verifying the complete configuration path from the CPU to the controller's Message RAM.

The following milestones were completed:

```text
✓ RTEMS application execution on BBB
✓ DCAN base address verification
✓ DCAN clock enable
✓ Controller register access
✓ INIT / CCE configuration sequence
✓ Bit Timing Register access
✓ TX Message Object configuration
✓ IF1 → Message RAM transfer
✓ Message RAM → IF1 readback transfer
✓ Message Object verification
```

In addition to DCAN-specific knowledge, I gained a deeper understanding of several important embedded systems concepts:

```text
Memory-mapped I/O
Register addressing
Clock gating
Synchronous digital logic
D Flip-Flops
CAN bit timing
Time Quanta (TQ)
Message RAM architecture
Interface Registers (IF1/IF2)
Acceptance masks
```

One of the most valuable lessons was learning that CAN controllers are much more than simple transmit and receive registers. The AM335x D-CAN architecture uses Interface Registers and Message Objects as an intermediate layer between software and hardware, which introduced me to a more sophisticated peripheral design than I had previously encountered.

By the end of the week, I successfully verified the complete data path:

```text
CPU
    ↓
IF1 Registers
    ↓
Message RAM
    ↓
Message Object 1
    ↓
IF1 Registers
    ↓
CPU
```

The readback results matched the original configuration exactly, confirming that the controller configuration mechanism is functioning correctly.

Overall, Week 1 established a solid foundation for future driver development. The next phase of the project will focus on actual CAN communication by requesting frame transmission, connecting external CAN hardware. And eventually integrating the implementation into the RTEMS CAN framework.


## May 25
* Read the Technical Reference Manual to understand the D-CAN.
* Build a repo in Gitlab. Refer to the RTEMS style, and CTU CAN FD structure.
* [Build hello app to test](https://docs.rtems.org/docs/main/user/start/app.html)
* [Convert ELF Executable and Linkable Format to U-Boot image](https://docs.rtems.org/docs/main/user/bsps/arm/beagle.html)

## May 26

RTEMS BBB TFTP Boot Workflow Simplified Main Flow version and details can be seen in this blog. Unfold week_1 tab will see details.  
* [Booting via Network](https://docs.rtems.org/docs/main/user/bsps/arm/beagle.html)


## May 27
* DCAN initial test, try and learn the registers in DCAN. I was reading the Technicial Reference Mannual for several days and feel confused.Until the mentor said try it and commit it every day in Gitlab. That is a good way to learn. Do it and learn from doing. Mere reading helps little. Avoid **Paralysis by analysis** Unfold week_1 tab will see details.

## May 28,29
* Bringing Up the AM335x D-CAN Controller on RTEMS.
* Register Access, Clock Enable, and Message Object Configuration.
* From Register Access to Message Object Configuration on BeagleBone Black.
* Combine with Technical Reference Manual