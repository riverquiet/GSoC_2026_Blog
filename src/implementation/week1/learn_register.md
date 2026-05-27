# May 26, 27

# Initial D-CAN Address Read

## Goal

The goal of this experiment was to verify that an RTEMS application running on the BeagleBone Black could successfully access the AM335x D-CAN peripheral base addresses through MMIO-based register access.

---

## Background

The AM335x SoC maps hardware peripherals into memory address space.

The following D-CAN base addresses were used:

```text
DCAN0: 0x481CC000
DCAN1: 0x481D0000
```

These addresses were obtained from:

```text
soc_AM335x.h
```

and verified against the AM335x TRM memory map.

---

## Implementation

A simple RTEMS application was created which:

1. booted through U-Boot and TFTP
2. entered the RTEMS Init task
3. called `dcan_reg_test()`
4. printed the D-CAN peripheral base addresses

Relevant code:

```c
printf("DCAN0 base: 0x%08lx\n",
  (unsigned long) SOC_DCAN_0_REGS);

printf("DCAN1 base: 0x%08lx\n",
  (unsigned long) SOC_DCAN_1_REGS);
```

---

## Result

The UART output showed:

```text
DCAN0 base: 0x481cc000
DCAN1 base: 0x481d0000
```

This confirmed:

- RTEMS boot succeeded
- the Init task executed correctly
- `soc_AM335x.h` was included correctly
- D-CAN peripheral addresses matched the TRM

---

## Significance

This experiment established the first successful interaction between the RTEMS application and the AM335x D-CAN hardware address space.

This forms the foundation for future:

- clock enable
- register access
- D-CAN initialization
- message object configuration
- CAN TX/RX testing


# DCAN1 Clock Enable Test

## Goal

The goal of this experiment was to enable the DCAN1 module clock through the AM335x PRCM subsystem and verify successful access to D-CAN control and status registers.

---

## Background

Modern SoCs use clock gating to reduce power consumption.

Peripheral modules such as:

- UART
- SPI
- I2C
- CAN

are often disabled by default until their clocks are explicitly enabled.

Without enabling the peripheral clock, MMIO register access may fail and generate bus exceptions.

The DCAN1 clock control register is:

```text
CM_PER_DCAN1_CLKCTRL
```

located inside the PRCM clock module.

---

## Implementation

The following register write was used to enable the DCAN1 module clock:

```c
REG32(SOC_CM_PER_REGS + CM_PER_DCAN1_CLKCTRL) = 0x2;
```

where:

```text
0x2 = MODULEMODE_ENABLE
```

After enabling the clock, the application read:

- DCAN1 CTL register
- DCAN1 ES register

through MMIO access.

Relevant code:

```c
printf("DCAN1 CTL: 0x%08lx\n",
  (unsigned long) REG32(SOC_DCAN_1_REGS + DCAN_CTL));

printf("DCAN1 ES: 0x%08lx\n",
  (unsigned long) REG32(SOC_DCAN_1_REGS + DCAN_ES));
```

---

## Result

The UART output showed:

```text
CM_PER_DCAN1_CLKCTRL: 0x00030002
DCAN1 CTL: 0x00001401
DCAN1 ES: 0x00000207
```

This confirmed:

- DCAN1 clock enable succeeded
- PRCM register access worked correctly
- DCAN1 MMIO registers could be read successfully
- no RTEMS exception occurred

---

## TRM References

Relevant AM335x TRM sections:

- Memory Map
- PRCM / Clock Management
- CM_PER_DCAN1_CLKCTRL
- DCAN Control Register (CTL)
- DCAN Error and Status Register (ES)

---

## Significance

This experiment established successful interaction between:

```text
RTEMS
→ PRCM clock subsystem
→ DCAN1 hardware
→ MMIO register interface
```

This forms the basis for future:

- DCAN initialization mode control
- bit timing configuration
- interface register usage
- message object configuration
- CAN frame TX/RX