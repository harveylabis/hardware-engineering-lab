# TM4C123GH6PM and Cortex-M4 learning path

This course uses the EK-TM4C123GXL LaunchPad and C. We will learn direct register access first, then compare it with TI DriverLib.

Lessons will use short sentences and one main goal. Each will include a small code example, click-by-click CCS steps, a prediction, a hands-on task, and a short check. We will take the time needed to understand each idea before moving on.

The goal is to learn transferable concepts, not memorize one chip's register names. We will use the TM4C123 as a concrete example, then identify which ideas carry over and which details must be checked in another device's reference manual.

## Exercise format

Every lesson must contain at least five exercises. Each exercise must:

- Extend the main lesson instead of repeating the main demonstration.
- Have a clear, separate **Goal**.
- Have a complete **Solution**.
- Include a measurable CCS, hardware, or debugger check when possible.
- Use direct register or architecture concepts covered by that lesson.
- Keep any device or confidential information out of the exercise.

Use this structure:

~~~markdown
### Exercise 1 — Name

**Goal:** State what the learner must practice.

**Solution:**

```c
/* Solution code */
```

**Check:** State what to observe in CCS or on the hardware.
~~~

The device and board references already in this folder are:

- [TM4C123GH6PM data sheet](./tm4c123gh6pm.pdf)
- [EK-TM4C123GXL LaunchPad user's guide](./spmu296.pdf)

## How register addresses are written

Most addresses are calculated as:

```text
peripheral base address + register offset
```

The base address and offset both come from the TM4C123GH6PM data sheet. The offset is in bytes.

For example:

```text
Port F APB base  = 0x40025000
GPIODIR offset   = 0x400
PF direction     = 0x40025000 + 0x400 = 0x40025400
```

GPIO `GPIODATA` has address aliases. For one or more selected pins:

```text
alias address = GPIO base + (pin mask << 2)
```

For PF1, the pin mask is `0x02`, so `0x02 << 2 = 0x08`:

```text
0x40025000 + 0x08 = 0x40025008
```

## Starting point

The learner is comfortable with C and electronics, but is new to MCU firmware, pointer-based peripheral access, and board-level firmware debugging. Lessons will introduce one new idea at a time and distinguish what C guarantees from what the TM4C hardware does.

## Development environment

The computer is an Apple Silicon Mac with Homebrew, VS Code, and **TI Code Composer Studio 21.0.1** installed. CCS includes the TI Arm Clang compiler, and its installation contains a TM4C123GH6PM device-support file. `arm-none-eabi-gcc`, `arm-none-eabi-gdb`, and `openocd` are not currently on `PATH`. The EK-TM4C123GXL has an onboard ICDI debug interface. With the board connected, macOS reports a Texas Instruments **In-Circuit Debug Interface** USB device, confirming USB enumeration.

**Starting recommendation: use the installed CCS first.** This avoids adding tools before we need them and gives us a TI compiler and a supported TM4C target definition in the same environment. The TI [CCStudio for VS Code page](https://www.ti.com/tool/CCSTUDIO-VSCODE) also describes a GDB/OpenOCD workflow. OpenOCD publishes a [`ti-icdi.cfg` interface definition](https://github.com/openocd-org/openocd/blob/master/tcl/interface/ti-icdi.cfg) for TI's ICDI hardware, so an OpenOCD route is a possible fallback.

The local application, device files, and USB enumeration are promising, but do not yet prove that CCS can halt and debug the MCU reliably on this Mac. Before calling the setup complete, use CCS to connect, halt, read `DID0`, and resume. Do not install the separate GNU toolchain or OpenOCD unless CCS cannot complete that test. If the supplied USB-A cable cannot connect to the Mac, use a USB-C data adapter or cable.

## Flexible lesson roadmap

Plan on one focused lesson each week, but the pace is flexible. We can repeat a topic or split a dense topic into smaller lessons. Lessons 1–6 are complete; Lesson 7, GPIO interrupts, is in progress. This roadmap is a checklist, not a deadline.

| Sequence | Focus | Topics and evidence of learning |
|---|---|---|
| Lessons 1–6 | Embedded C and GPIO foundations — complete | Memory-mapped registers, addresses, masks, `volatile`, GPIO output and input, active-low buttons, polling, and button-driven LED behavior |
| Lesson 7 | GPIO interrupts — in progress | Edge detection, GPIO interrupt flags, NVIC enable, vector-table handler, and a button-triggered LED action |
| Lessons 8–10 | Cortex-M4 and memory | R0–R15, SP/LR/PC, exception entry and return, Thumb-2 basics, memory map, Flash/SRAM, static/stack/heap storage, reset sequence, vector table, startup code, and linker map |
| Lesson 11 | NVIC and interrupt behavior | Pending and enable state, priorities, nesting, global masking, and interrupt latency; explain and measure when practical |
| Lesson 12 | GPIO details and debouncing | Pin multiplexing, pull-up and pull-down resistors, protected pins, and a timed switch-debounce method |
| Lessons 13–14 | SysTick and general-purpose timers | Periodic and one-shot timing, timer interrupts, input capture, output compare, and timing measurements |
| Lesson 15 | UART | Send and receive data, then build a small command interface and inspect serial traffic |
| Lesson 16 | SSI/SPI | Connect to a peripheral and inspect clock/data signals with a logic analyzer or suitable ADALM2000 setup |
| Lesson 17 | I2C | Read and write a peripheral and inspect bus transactions |
| Lesson 18 | ADC | Configure a conversion, sample a signal, and compare measured values with expected results |
| Lesson 19 | PWM | Generate and measure a PWM output; relate timer settings to frequency and duty cycle |
| Lessons 20–22 | Watchdog, uDMA, and low-power modes | Learn what each feature does and complete small, observable experiments |
| Project lessons | Integration | Build a sensor acquisition system, a UART command interface, an SPI/I2C sensor interface, timer-triggered ADC acquisition, and a multichannel monitor with Python visualization |

Embedded C topics already introduced—pointers, bitwise operations, `volatile`, and integer types—will be revisited where useful. The memory lessons will make Flash, SRAM, and static/stack/heap storage explicit. Each project will include its goal, connections, source, expected and actual measurements, debugging notes, and lessons learned.

## Lessons

- [Lesson 1 — Embedded C and memory-mapped registers](lesson-01-memory-mapped-registers.md)
- [Lesson 2 — Keep part of a value](lesson-02-bit-mask.md)
- [Lesson 3 — Set and clear selected bits](lesson-03-set-and-clear-bits.md)
- [Lesson 4 — Turn on the red LED](lesson-04-gpio-red-led.md)
- [Lesson 5 — Read a GPIO button](lesson-05-gpio-button.md)
- [Lesson 6 — Control the red LED with SW1](lesson-06-button-controls-led.md)
- [Lesson 7 — Respond to a GPIO interrupt](lesson-07-gpio-interrupts.md)
- [Lesson 8 — Meet the Cortex-M4 registers](lesson-08-cortex-m4-registers.md)
