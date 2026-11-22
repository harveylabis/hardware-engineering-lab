# Lesson 8 — Meet the Cortex-M4 registers

## Today's goal

Recognize the Cortex-M4's core registers in CCS and understand the jobs of `SP`, `LR`, and `PC`.

These are registers inside the CPU. Unlike GPIO registers, they do not have peripheral memory addresses.

## Before you run

When C calls a function, the CPU must know where to return afterward.

Predict: Which register do you think holds the return address?

## The core registers

The Cortex-M4 has registers named `R0` through `R15`. Some have common names and special roles:

| Register(s) | Common role |
|---|---|
| `R0`–`R12` | General-purpose registers. The compiler uses these to work with values. |
| `R13` (`SP`) | Stack Pointer. It points into the active stack. |
| `R14` (`LR`) | Link Register. A function call normally places its return address here. |
| `R15` (`PC`) | Program Counter. It identifies the next instruction to execute. |
| `xPSR` | Program status. It includes condition flags and processor state. |

`R13` has two possible stack pointers on Cortex-M: the Main Stack Pointer (MSP) and Process Stack Pointer (PSP). The active one is shown as `SP`. We will study the two-stack behavior later.

For ordinary function calls, the Arm calling convention also uses `R0`–`R3` for the first arguments and a return value. The compiler follows this convention; you do not need to move the values into registers yourself.

The stack grows toward lower memory addresses on this processor. The exact stack value depends on the program and compiler, so do not expect a fixed number.

## The program

Build this small example in CCS:

```c
#include <stdint.h>

volatile uint32_t first_value;
volatile uint32_t second_value;
volatile uint32_t result;

uint32_t add_values(uint32_t left, uint32_t right)
{
    return left + right;
}

int main(void)
{
    first_value = 7u;
    second_value = 5u;
    result = add_values(first_value, second_value);

    while (1)
    {
    }
}
```

The result should be `12`. The function call gives CCS a useful place to inspect the core registers.

## CCS steps

1. Put the project in its **Debug** build configuration and build it.
2. Start **Debug As → Code Composer Debug Session**.
3. Open the **Registers** view (in CCS, use **View → Registers** or **Window → Show View → Registers**, depending on the layout).
4. In the Registers view, expand the **CPU/Core Registers** group. Look for `R0`–`R15` and `xPSR`. CCS may label `R13`, `R14`, and `R15` with their aliases `SP`, `LR`, and `PC`.
5. Set a breakpoint on the first statement inside `add_values`.
6. Resume. The debugger should stop when `add_values` is called.
7. Inspect `SP`, `LR`, and `PC`. Step through the return and watch execution go back to `main`.

At the call boundary, the first two arguments are passed in `R0` and `R1` by the Arm calling convention. A source-level breakpoint may stop after the function prologue has moved them, so use the Variables view to inspect `left` and `right`. To see the arguments in `R0` and `R1`, stop at the function's first machine instruction in the Disassembly view. `LR` holds the return address. When halted, the debugger's `PC` normally identifies the instruction that will execute next.

## Expected result

- `first_value` is `7`.
- `second_value` is `5`.
- `result` becomes `12`.
- At the call boundary, `R0` and `R1` carry the arguments `7` and `5`.
- `LR` provides the address to return to in `main`.
- `SP` points into SRAM and may change as the function uses stack space.

Register values besides the argument values and final result can vary with compiler settings. That is normal.

## Questions we discussed

### Are these registers the same on Cortex-M3 and Cortex-M33?

**Question:** Can I apply this register knowledge to the Cortex-M3 and Cortex-M33 MCUs used at work?

**Answer:** Yes. The core register names and basic roles—`R0`–`R15`, including `SP`, `LR`, and `PC`, plus `xPSR`—are common across Cortex-M3, M4, and M33. This is useful knowledge to carry between projects.

The cores still have differences. Cortex-M4 adds DSP instructions and may include an FPU; Cortex-M33 adds TrustZone security features. Those can add instructions or extra register state to inspect. Debugger labels and peripheral registers also depend on the specific core implementation and MCU. When you change chips, keep the core concepts and check that device's reference manual and debug view for details.

### How deeply should I learn `R0`–`R15`?

**Question:** Do I need to memorize or deeply study every core register now?

**Answer:** Not yet. For writing C firmware, first learn to recognize what the registers are for:

- `R0`–`R3` often carry function arguments and return values.
- `SP` points to the active stack.
- `LR` helps a function return to its caller.
- `PC` identifies the next instruction to execute.
- `xPSR` holds status and condition information.

You do not need to memorize changing register values or write assembly to use peripherals in C. We will go deeper when it helps explain function calls, startup, interrupts, faults, or debugger behavior. For now, being able to find these registers in CCS and describe their basic roles is enough.

### What is the stack?

**Question:** I understand `LR` and `PC`, but what does the stack pointer point to?

**Answer:** The stack is a part of SRAM used as temporary working space. Think of it like a pile of trays: the most recently added item is usually the first one removed. The CPU uses the stack to save information it needs temporarily, such as a function's saved registers or local data.

`SP` is the Stack Pointer. It marks the active top of the stack. When code reserves stack space, `SP` moves; when that space is released, `SP` moves back. On Cortex-M, the stack normally grows toward lower memory addresses. The exact value and movement depend on what the compiler generates.

`LR` and the stack have related but different jobs: a function call puts its return address in `LR`; code may save `LR` on the stack if it needs to preserve it. For an interrupt, the Cortex-M automatically saves key CPU state on the active stack. We will trace that in the next lesson.

It is called a **stack** because it follows the same last-in, first-out idea as a pile of trays: add an item on top, then remove the newest item first. The MCU's stack is stored in RAM; it is not a physical pile.

### What should I remember about `R0`–`R15`?

**Question:** Is this the important summary for now?

**Answer:** Yes, with a couple of refinements:

- The Cortex-M4 has 16 core registers named `R0`–`R15`. `R0`–`R12` are general-purpose. `R13` is `SP`, `R14` is `LR`, and `R15` is `PC`.
- `SP` points to the active top of a temporary stack area in SRAM. The stack follows last-in, first-out behavior; it can hold saved state, return information, and temporary data.
- For an ordinary function call, `LR` holds the return address. An interrupt uses a special exception-return value in `LR`; we will cover that later.
- `PC` identifies the next instruction to execute. When the CPU is halted at a breakpoint, the CCS disassembly view shows where execution will resume. Some instructions that read `PC` observe a pipeline-adjusted value, so that value is not always the address shown for the current instruction.

There are also other Cortex-M registers, such as `xPSR` and interrupt-control registers. You do not need to memorize them now. These roles are enough for this lesson and to prepare for the next one.

## Your notes

- My prediction about the return address:
- What did CCS show for `R0` and `R1` at function entry?:
- Did execution return to `main` after the function?:
- What did I notice about `SP`?:
- What is still unclear?:

## Short check

1. Which register points to the active stack?
2. Which register normally holds a function's return address?
3. Which register tracks the current instruction?
4. Are the core registers peripheral memory-mapped registers?

## Beginner-friendly resources

- [Embedded Systems — Shape the World](https://users.ece.utexas.edu/~valvano/Volume1/E-Book/) by Jonathan Valvano and Ramesh Yerraballi is a free, book-like introduction that uses the Tiva C LaunchPad. For this course, try Chapter 2, “Fundamental Concepts,” and Chapter 12, “Interrupts.” It is an older course, so use this TM4C123 course and the current data sheet to confirm device-specific details.
- In the [TM4C123GH6PM data sheet](./tm4c123gh6pm.pdf), read Chapter 2.5, “Exception Model,” and the “Vector Table” section around printed pages 101–106. Read a few pages at a time; it is a reference manual, not a beginner textbook.
- For later reference, the Arm *Cortex-M4 Devices Generic User Guide* (DUI 0553) describes the processor registers, exceptions, and instruction set. It is more detailed, so you do not need to read it cover to cover.

## Exercises

### Exercise 1 — Follow the Program Counter

**Goal:** See that `PC` changes as the CPU executes instructions.

**Solution:** Set a breakpoint on `result = add_values(first_value, second_value);`. Resume and note `PC`. Step into the function, then step through a few instructions while watching `PC`.

**Check:** `PC` should change as you step. Its exact addresses depend on the build and compiler.

### Exercise 2 — Change the function arguments

**Goal:** Observe new argument values passed in `R0` and `R1`.

**Solution:** Change the assignments in `main`:

```c
first_value = 20u;
second_value = 22u;
```

Rebuild, restart debugging, and stop at `add_values`. Check the Variables view for `left` and `right`. To inspect the argument registers, stop on the function's first machine instruction in the Disassembly view, before the function prologue.

**Check:** The parameters should be `20` and `22`; at the call boundary, `R0` and `R1` should carry those values. `result` should become `42`.

### Exercise 3 — Observe the Link Register

**Goal:** See how a function call provides a return address.

**Solution:** Stop at the first statement in `add_values` and record `LR`. Step over the return statement and continue until the next statement in `main`.

**Check:** Execution should return to `main` after the call. `LR` at function entry should identify the return location; the exact address depends on the build.

### Exercise 4 — Observe the Stack Pointer

**Goal:** See where the active stack is and how a function can use stack space.

**Solution:** Add a local variable to `add_values`:

```c
uint32_t temporary = left + right;
return temporary;
```

Set a breakpoint at the first statement in the function and inspect `SP`. Step through the function and inspect it again.

**Check:** `SP` should point into SRAM. It may change when the function creates stack space; exact values and changes depend on compiler code generation.

### Exercise 5 — Find the Thumb state bit

**Goal:** See that the Cortex-M4 executes Thumb instructions.

**Solution:** Stop at a breakpoint and inspect `xPSR` in the Registers view. If CCS shows the raw value, examine bit 24 (the T bit).

**Check:** The T bit should be `1`. Cortex-M processors execute Thumb instructions; the TM4C123's Cortex-M4 supports Thumb-2.

## Next lesson

Next, we will follow a hardware interrupt into an exception handler and see how the Cortex-M4 uses the stack and vector table.
