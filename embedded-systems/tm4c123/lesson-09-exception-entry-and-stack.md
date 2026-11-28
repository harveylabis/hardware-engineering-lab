# Lesson 9 — What the CPU does when an interrupt arrives

## Today's goal

Follow a GPIO interrupt from the Port F hardware to the handler, and see how the Cortex-M saves and restores CPU state.

We will build on the working GPIO interrupt from [Lesson 7](lesson-07-gpio-interrupts.md). Keep that project and startup-vector change.

## Before you run

When SW1 triggers an interrupt, the CPU must remember where the main program stopped so it can resume later.

Predict: Does the CPU save the interrupted program's `PC` somewhere before it runs the handler?

## The path into an interrupt

For a press of SW1, the sequence is:

1. PF4 detects a falling edge and sets its interrupt flag.
2. The GPIO interrupt mask and NVIC settings allow the request to reach the Cortex-M.
3. The Cortex-M finishes the current instruction and saves basic CPU state on the active stack.
4. It reads the Port F handler address from the vector table and enters `GPIOPortFIntHandler`.
5. The handler clears the GPIO flag and handles the event.
6. The handler returns. The CPU restores the saved state and resumes the interrupted code.

The CPU does not call the handler like an ordinary C function call. Exception entry and return are processor behavior.

## The basic exception stack frame

For an interrupt from ordinary code, the Cortex-M automatically saves these eight 32-bit values:

| Saved value | Why it is saved |
|---|---|
| `R0`–`R3` | Working registers that the interrupted code may need |
| `R12` | Another working register |
| `LR` | The interrupted code's link register |
| `PC` | Where the interrupted code should resume |
| `xPSR` | Processor status and state |

That is 8 words, or 32 bytes. Depending on stack alignment, the processor may also reserve one extra word. A Cortex-M4F may save additional floating-point state if floating-point context is active; we are not using floating point in this exercise.

After stacking, `SP` points to the basic frame. At the first instruction of the handler, if the interrupted program was using MSP, the frame is at the top of MSP. If Thread mode was using PSP, the frame is saved on PSP, while the handler itself runs using MSP. The startup code in our simple program uses MSP.

The frame is laid out in memory like this, from the address in `SP` toward higher addresses:

```text
SP + 0x00   saved R0
SP + 0x04   saved R1
SP + 0x08   saved R2
SP + 0x0C   saved R3
SP + 0x10   saved R12
SP + 0x14   saved LR
SP + 0x18   saved PC
SP + 0x1C   saved xPSR
```

The stack grows toward lower addresses when space is reserved. The frame values themselves are shown above in increasing address order.

## The vector-table entry

The data sheet assigns Port F IRQ number 30. Cortex-M exception numbers reserve 0–15 for processor exceptions, so the vector number for Port F is:

```text
vector number = IRQ number + 16
              = 30 + 16
              = 46

vector entry offset = 46 * 4 bytes = 184 bytes = 0xB8
```

With the vector table at address `0x00000000`, its Port F entry is at `0x000000B8`. That entry holds the handler address. Your startup file's entry at that position names `GPIOPortFIntHandler`.

The vector table may be relocated on some systems. In that case, add the offset `0xB8` to the vector table's base address instead of assuming the base is zero.

## `LR` during exception handling

During an ordinary C function call, `LR` holds a return address. During exception handling, the processor puts a special **EXC_RETURN** value in `LR`. The handler's return sequence uses it to tell the processor to restore the saved state and return from the exception.

For this program's simple Thread-mode/MSP case, CCS may show `0xFFFFFFF9` in `LR` at the start of the handler. This value is an exception-return code, not a normal program address. Other stack or floating-point configurations can use a different EXC_RETURN value.

Do not change or use this value as an ordinary function pointer. The compiler-generated handler return takes care of it.

## CCS steps

1. Use the working Lesson 7 SW1 interrupt project. Confirm the startup vector still points to `GPIOPortFIntHandler`.
2. At file scope, add this main-program counter. Increment it in the empty main loop:

   ```c
   volatile uint32_t main_count;

   // Inside main:
   while (1)
   {
       main_count++;
   }
   ```

3. Build in the **Debug** configuration and start a debug session.
4. Add `main_count`, `led_state`, and `interrupt_count` to the **Watch** view.
5. Set a breakpoint on the first machine instruction of `GPIOPortFIntHandler`. If you set a C source breakpoint, the compiler may execute a function prologue first.
6. Resume and press SW1. At the breakpoint, inspect `PC`, `SP`, and `LR` in the Registers view.
7. If `LR` is `0xFFFFFFF9`, recognize it as EXC_RETURN for this simple setup. Resume and check that `main_count` continues increasing.

Do not expect a fixed `SP` value. It depends on the linker setup, compiler, and debug build.

### The four CCS debug views

| View | What it shows |
|---|---|
| **Threads** | Execution contexts known to the debugger. A bare-metal program usually has one main execution context; an RTOS project may show several tasks. Interrupts are processor exceptions, not normally separate application threads. |
| **Call Stack** | The chain of active function calls for the selected context, such as `main` calling another function. An interrupt handler may appear while stopped in an ISR; details depend on the debugger and unwind information. |
| **Variables** | Local variables and function arguments for the selected stack frame. Select a different frame in **Call Stack** to inspect that function's locals. |
| **Watch** | Variables or expressions that you choose to monitor, including globals such as `main_count`. This is the view meant by “Expressions” in some CCS instructions. |

To add the Lesson 9 variables, start a debug session, select **Watch**, and use its **Add Expression** control (often a `+` icon or right-click menu). Enter `main_count`, `led_state`, and `interrupt_count` as separate expressions. If the values do not update while the CPU is running, halt at a breakpoint to inspect them. A watched name must be a real variable available in the loaded program; a preprocessor macro is not a debugger variable.

## Expected result

- Pressing SW1 reaches the vector-table entry for Port F and stops in `GPIOPortFIntHandler`.
- The processor saves the interrupted context before the handler runs.
- The handler clears the peripheral interrupt flag and updates the LED.
- After the handler returns, `main_count` continues to increase.

## Questions we discussed

### What is the main idea of this lesson?

**Question:** Is the main idea that `LR` becomes `0xFFFFFFF9` when an ISR is triggered?

**Answer:** Not quite. `0xFFFFFFF9` is what we observed for this program's exception-return setup; other modes or stack configurations can produce a different EXC_RETURN value. The main idea is the whole sequence:

1. A peripheral interrupt reaches the NVIC.
2. The Cortex-M saves a basic CPU context—including the interrupted `PC`—on the active stack.
3. The vector table provides the ISR address, and the CPU runs the handler.
4. `LR` contains an EXC_RETURN code so the processor knows how to leave the handler.
5. On exception return, the processor restores the saved context and continues the interrupted code.

For now, remember that the saved `PC` is on the stack, while `LR` carries the special return instruction/code. You do not need to memorize `0xFFFFFFF9`.

### Why does the vector contain `0x26D` when the handler starts at `0x26C`?

**Question:** CCS shows `0x000000B8` contains `0x0000026D`, while the first handler instruction is at `0x0000026C`. Why are they different?

**Answer:** In a Cortex-M vector table, bit 0 of a handler address is set to indicate Thumb code. The actual instruction address is `0x26C`; the vector stores `0x26D` to include the Thumb-state bit. It is not a one-byte shift into the handler. This is the expected representation for a Cortex-M handler.

**Observed on this setup:** The Port F vector entry at `0x000000B8` contains `0x0000026D`, and CCS disassembles the handler starting at `0x0000026C`.

### Why does the CCS Memory view advance by `0x20` per row?

**Question:** Why does a row starting at `0x200001E0` show eight 32-bit values, and the next row start at `0x20000200`?

**Answer:** CCS is displaying eight words per row for readability. Each 32-bit word is 4 bytes, so one row covers `8 * 4 = 32` bytes, or `0x20`. Therefore, the next row starts at `0x200001E0 + 0x20 = 0x20000200`. This is a display layout, not a special memory boundary.

The basic Cortex-M exception frame is also eight 32-bit words. If the handler's `SP` is `0x200001E0` before its function prologue changes the stack, those eight displayed words can be the saved exception frame. Check `SP` in the Registers view to confirm the frame's starting address.

### Does a 32-bit CPU have addresses between the word addresses?

**Question:** If the Memory view shows `0x200001E0` and then `0x200001E4`, do addresses such as `0x200001E1` exist?

**Answer:** Yes. Cortex-M memory is byte-addressed: each address identifies one byte. A 32-bit word starting at `0x200001E0` occupies bytes `0x200001E0` through `0x200001E3`. The next aligned 32-bit word starts at `0x200001E4`. CCS is displaying the starting address of each 32-bit word, so the byte addresses inside each word are not shown as separate columns.

### Does a 32-bit CPU always access four bytes at once?

**Question:** Since the Cortex-M is a 32-bit CPU, does it always access a full four-byte word?

**Answer:** No. “32-bit” describes the processor's core register width and common data path, not every memory access. The instruction determines the access size: a word load/store accesses 32 bits (4 bytes), a halfword access is 16 bits (2 bytes), and a byte access is 8 bits (1 byte). C types and the generated instructions determine which size is used. The exception frame uses 32-bit words because it saves 32-bit core registers.

**Follow-up:** In this course, “32-bit CPU” mainly means the Cortex-M core's general-purpose registers (`R0`–`R15`) are 32 bits wide and the core performs 32-bit operations. It does not mean every memory access must transfer four bytes.

**Another follow-up:** `0x1` is the number one written in hexadecimal; by itself, it does not mean one byte. Because memory addresses are byte-addressed, adding `1` to an address moves one byte. The data type or instruction determines how many bytes of data are stored or accessed.

**Observed on this setup:** With the frame beginning at `0x200001E0`, the saved `PC` word at offset `0x18` was `0x00000322`. This is the code address where execution should resume after the ISR. To identify the exact instruction, navigate the CCS Disassembly view to `0x00000322`; the mapping to a C source line depends on the compiled program.

**Follow-up observation:** CCS disassembled `0x00000322` as `mov.w r0, #0x40000000`. This loads `PORTF_IRQ_MASK` (`1u << 30`), the Port F NVIC enable mask. If this is truly the stacked `PC`, the interrupted code was around NVIC setup, which would be unexpected for a button press after the main loop was running.

Check that `SP` was captured at the handler's first machine instruction, before the C function prologue changes it. The `SP + 0x18` calculation identifies the saved `PC` only when `SP` points to the start of the hardware exception frame. A source-level breakpoint may stop later, after the compiler has used the stack. In CCS, set a breakpoint on the handler's first instruction in Disassembly, then inspect `SP` and the frame. Also check the source line and the instruction after `0x322` to see whether it is part of the NVIC setup sequence.

### Does the CPU save the interrupted `PC`?

**Question:** Does the CPU save the interrupted program's `PC` before it runs the handler?

**Answer:** Yes. On exception entry, the Cortex-M automatically stacks the interrupted context, including `PC` and `xPSR`. The saved `PC` tells the CPU where to resume after the handler returns.

### Does each GPIO pin have its own handler?

**Question:** Can each GPIO pin have a separate interrupt handler?

**Answer:** On this TM4C123, GPIO interrupts are grouped by port. Each port has one interrupt vector and handler entry for all of its pins. For example, any enabled Port F pin that triggers an interrupt enters the Port F handler. The handler reads `GPIO_PORTF_MIS` to see which pin or pins caused it, then handles those bits separately. Multiple pending pins can be reported together in the same status value.

### What are Threads, Call Stack, Variables, and Watch in CCS?

**Question:** What do these four debug views show, and where should I add `main_count`?

**Answer:** **Threads** lists execution contexts, **Call Stack** shows the active function chain, **Variables** shows values for the selected function frame, and **Watch** lets you enter variables or expressions to monitor. Add `main_count`, `led_state`, and `interrupt_count` in **Watch**; some CCS versions or instructions call this the Expressions view.

### Why can't CCS read `PC` at the ISR breakpoint?

**Question:** CCS says, “Could not read register PC: execution state prevented access,” after SW1 stops at the handler breakpoint. What should I check?

**Answer:** A breakpoint should normally suspend the Cortex-M so its registers can be inspected. This message does not by itself mean the ISR code is wrong. First check the debugger context:

1. Confirm the debug toolbar says the target is **Suspended** rather than **Running**. If it is running, click **Suspend** and wait for the halt to complete.
2. In **Threads**, select the halted Cortex-M execution context.
3. In **Call Stack**, select the top frame for `GPIOPortFIntHandler`, then refresh the **Registers** view.
4. Look for `PC` or its alias `R15`. If CCS still cannot read it, try a normal breakpoint in `main` to see whether the issue occurs only in the ISR.

If the target is suspended on the ISR frame and `PC` is still unavailable, capture the CCS Debug view, Call Stack selection, and Registers view for further diagnosis. CCS versions and debug probes can differ in how they present exception contexts.

**Observed on this setup:** Setting a breakpoint in `main` first, then resuming to the `GPIOPortFIntHandler` breakpoint, made `PC` visible in the Registers view.

**Observed during the lesson:** At the Port F handler after pressing SW1, `LR` showed `0xFFFFFFF9`. This is the Cortex-M's EXC_RETURN code for returning to Thread mode using MSP with the basic exception frame.

## Your notes

- Current understanding: when an interrupt occurs, the Cortex-M saves the basic interrupted context on the stack, runs the ISR, restores that context, and resumes the interrupted code.
- My prediction about the saved `PC`:
- What values did CCS show for `SP` and `LR` at handler entry?:
- Did `main_count` continue after the interrupt?:
- What is still unclear?:

## Short check

1. What does the CPU save automatically on exception entry?
2. Where does the CPU get the handler address?
3. What is the Port F vector number?
4. How is EXC_RETURN different from an ordinary `LR` return address?
5. What does the CPU do after the handler returns?

## Exercises

### Exercise 1 — Locate the Port F vector entry

**Goal:** Find the vector-table slot used for Port F.

**Solution:** Calculate the vector number and byte offset:

```text
IRQ 30 + 16 = vector 46
vector 46 * 4 bytes = offset 0xB8
```

In the CCS Memory Browser, inspect address `0x000000B8` while the vector table is at address zero.

**Check:** The word at `0xB8` should contain the Port F handler address (possibly with bit 0 set to indicate Thumb code). It should not be the address of `IntDefaultHandler`.

### Exercise 2 — Prove that main resumes

**Goal:** Confirm that the CPU returns to the interrupted program after the ISR.

**Solution:** Keep `main_count++` in the main loop. Run the program and watch the counter. Press and release SW1 while the program runs.

**Check:** The counter should continue increasing after each interrupt. The handler should not permanently replace or restart `main`.

### Exercise 3 — See that both buttons use the same port vector

**Goal:** Verify that SW1 and SW2 can be separate pin sources handled by one Port F vector.

**Solution:** Starting from Lesson 7 Exercise 5, configure PF0/SW2 as a second falling-edge interrupt source and keep both pin bits enabled in `GPIO_PORTF_IM_R`. In the handler, snapshot `GPIO_PORTF_MIS_R` and test `status & SW1` and `status & SW2` separately.

**Check:** A press of either button should enter the same `GPIOPortFIntHandler`. The status snapshot should identify PF4 (`0x10`) for SW1 or PF0 (`0x01`) for SW2.

### Exercise 4 — Compare ordinary return with exception return

**Goal:** Distinguish an ordinary function return address from EXC_RETURN.

**Solution:** Add a small function and call it from `main`:

```c
volatile uint32_t helper_result;

uint32_t helper(uint32_t value)
{
    return value + 1u;
}

// Add this line in main, before the infinite loop:
helper_result = helper(10u);
```

Set a breakpoint at the first instruction of `helper`, then inspect `LR`. Next, set a breakpoint at the first instruction of `GPIOPortFIntHandler` and inspect `LR` there.

**Check:** In `helper`, `LR` should identify a code return location. In the interrupt handler, `LR` should be an EXC_RETURN code, commonly `0xFFFFFFF9` in this simple setup.

### Exercise 5 — Inspect the saved PC

**Goal:** Find the interrupted program's saved return location in the exception frame.

**Solution:** Stop at the handler's first machine instruction, before its compiler prologue. Read `SP` from the Registers view and inspect eight 32-bit words starting at that address in the Memory Browser. The saved `PC` is the word at `SP + 0x18`.

**Check:** The saved `PC` should point into the main program near where the interrupt occurred. The exact address depends on the build and the point at which SW1 was pressed.

## Next lesson

Next, we will study the NVIC in more detail: pending interrupts, priorities, and what happens when multiple interrupts request service.
