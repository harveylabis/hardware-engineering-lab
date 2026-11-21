# TM4C123GH6PM and Cortex-M4 learning path

This course uses the EK-TM4C123GXL LaunchPad and C. We will learn direct register access first, then compare it with TI DriverLib.

Lessons will use short sentences and one main goal. Each will include a small code example, click-by-click CCS steps, a prediction, a hands-on task, and a short check. We will take the time needed to understand each idea before moving on.

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

## Weekly plan

Plan on one focused lesson each week. We can slow down or repeat a week whenever needed.

| Week | Focus | Evidence of learning |
|---|---|---|
| 1 | Embedded C, memory addresses, and memory-mapped registers | Read a documented read-only register in the debugger and explain the access |
| 2 | Pointers, bitwise operations, `volatile`, and C integer types | Decode and update selected bit fields without disturbing unrelated bits |
| 3 | Flash, SRAM, stack, static storage, startup code, and linker map | Locate code and variables in the memory map |
| 4 | Cortex-M4 registers, Thumb-2, reset/vector table, exceptions, NVIC | Trace reset into `main` and inspect core state at a breakpoint |
| 5 | GPIO clock gating, digital enable, direction, data, and onboard LED | Configure and control the RGB LED using direct register access |
| 6 | GPIO input, pull resistors, pin mux, and GPIO interrupts | Read buttons and respond to an edge-triggered interrupt |
| 7 | SysTick and general-purpose timers | Generate a measured periodic event and inspect interrupt behavior |
| 8 | UART | Send and receive a simple command over a serial terminal |
| 9 | SSI/SPI and I2C | Read a peripheral and verify bus transactions with the ADALM2000 where suitable |
| 10 | ADC and PWM | Sample a signal and generate a measured PWM waveform |
| 11 | Watchdog, uDMA, and low-power modes | Explain when each is useful and implement a bounded, observable experiment |
| 12 | Integrating project | Build a timer-triggered acquisition path with a UART interface and Python visualization |

Each week's note will capture measured results and deviations from expected behavior. The final project can be expanded after the fundamentals are solid.

## Lessons

- [Lesson 1 — Embedded C and memory-mapped registers](lesson-01-memory-mapped-registers.md)
- [Lesson 2 — Keep part of a value](lesson-02-bit-mask.md)
- [Lesson 3 — Set and clear selected bits](lesson-03-set-and-clear-bits.md)
- [Lesson 4 — Turn on the red LED](lesson-04-gpio-red-led.md)
- [Lesson 5 — Read a GPIO button](lesson-05-gpio-button.md)
