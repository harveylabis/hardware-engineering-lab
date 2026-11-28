# Lesson 10 — The memory map: Flash and SRAM

## Today's goal

Tell whether an address refers to Flash, SRAM, or a peripheral register.

We will use addresses you have already seen. No new firmware is needed.

## Before you begin

You saw these addresses while debugging:

- `0x0000026C` — an instruction in `GPIOPortFIntHandler`
- `0x200001E0` — the stack frame in SRAM
- `0x400FE608` — `SYSCTL_RCGCGPIO`

Predict: Which address is Flash, which is SRAM, and which is a peripheral?

## What is a memory map?

Think of the address space as byte-sized numbered locations. The **memory map** is the device's chart that tells the CPU what responds to each address range.

An address does not always point to ordinary RAM. Depending on the range, it can select Flash, SRAM, a peripheral register, or a reserved area. Reserved areas are not usable storage.

## Three regions to recognize

| Region | TM4C123GH6PM address range | What it is for |
|---|---|---|
| On-chip Flash | `0x00000000`–`0x0003FFFF` | Firmware code and constant data |
| On-chip SRAM | `0x20000000`–`0x20007FFF` | Working data, stack, and runtime variables |
| Peripheral blocks | Device-defined ranges around `0x40000000`; some addresses are reserved | Control and status registers for hardware |

These are byte addresses. The Flash range contains `0x40000` bytes, or **256 KiB**. The SRAM range contains `0x8000` bytes, or **32 KiB**. The TM4C123GH6PM data sheet lists these ranges in Table 2-4, printed page 92.

### Flash

Flash is non-volatile: its contents normally remain when power is removed. The debugger showed your handler instruction at `0x0000026C`, which is inside the Flash range.

Firmware is usually programmed into Flash. Changing Flash requires a special erase/program process; it is not written like an ordinary C variable. We will not program Flash in this lesson.

### SRAM

SRAM is working memory. The Cortex-M uses it for the stack and for runtime data. The exception frame you inspected at `0x200001E0` is in the SRAM range.

SRAM is volatile: do not expect its contents to survive power being removed. On reset, startup code also initializes some variables. We will study that startup process later.

### Peripheral registers

Peripheral addresses select hardware registers, not ordinary storage. For example, `0x400FE608` selects `SYSCTL_RCGCGPIO`. Writing its bits asks the system-control hardware to enable peripheral clocks.

This is why a register access can change hardware behavior even though C code appears to read or write a memory address.

## What does CCS's `R` or `R|W` permission mean?

The **Memory Map** view describes which kinds of accesses the debugger permits for a region:

- `R` means the debugger permits reads but not ordinary direct writes.
- `R|W` means the debugger permits reads and writes to that region.

That does not mean every peripheral register is safely writable. Individual registers can be read-only, write-only, write-one-to-clear, or have other special behavior. Check the data sheet before writing a peripheral register.

Flash commonly appears as read-only in the debugger's memory map because it cannot be changed by an ordinary memory write. Firmware can still program or erase Flash by using the Flash Control module and following its required sequence. This is different from writing a normal SRAM variable.

## Can I keep a value through reset or power-off?

Yes. The TM4C123GH6PM has **2 KB of EEPROM**, intended for small settings or records that must persist when power is removed. Firmware accesses it through the EEPROM controller. Flash can also hold persistent data, but the firmware must reserve space and use the Flash controller; never write arbitrary values over the program image.

Both storage types have write limits and device-specific rules. For frequently updated settings, the write strategy matters. `volatile` does not make a value persistent; it only affects how the compiler handles accesses while the program runs.

For this lesson, just remember the distinction:

- **SRAM:** ordinary runtime variables; not retained through power loss.
- **EEPROM or reserved Flash:** can retain data through power loss when written with the proper controller sequence.

We will leave the actual EEPROM/Flash programming exercise for later.

### Is EEPROM included in every Cortex-M?

**Question:** Is EEPROM always present in a Cortex-M microcontroller?

**Answer:** No. Cortex-M describes the processor core, not the chip's memory or peripheral selection. The silicon vendor and specific MCU determine whether it has internal EEPROM, Flash only, or another type of nonvolatile memory. This TM4C123GH6PM has 2 KB of internal EEPROM. A design without internal EEPROM can sometimes use a reserved Flash area or an external EEPROM connected over a bus such as I2C or SPI.

## Your CCS check

Use your existing Lesson 9 debug project:

1. Start a debug session and halt the CPU.
2. In **Watch**, enter `&main_count`. It should show the variable's address.
3. Check that the address is inside `0x20000000`–`0x20007FFF`. This confirms that this global runtime variable is in SRAM.
4. Open **Memory Browser** and enter `0x0000026C`. This address is in Flash, where the handler instruction was located.
5. Remember that `0x400FE608` is in peripheral space, not Flash or SRAM.

The Memory Browser shows bytes or words at the selected address. The address range tells you what region you are looking at; the view's display format does not change the region.

## The simple model

For now, remember:

- **Flash:** keeps the firmware when power is off.
- **SRAM:** holds the stack and working data while the program runs.
- **Peripheral addresses:** let the CPU communicate with hardware registers.

You do not need to memorize every address. Learn to consult the memory map and classify an address range.

## Questions we discussed

Add your questions and observations here as we work through this lesson.

## Your notes

- My prediction:
- Address of `main_count`:
- Which range contains the handler instruction at `0x0000026C`?:
- Which range contains the stack frame at `0x200001E0`?:
- What is still unclear?:

## Short check

1. Which region keeps its contents when power is removed?
2. Where does the stack live on this TM4C123?
3. Is `0x400FE608` an SRAM address or a peripheral register address?
4. Which document tells you what an address range represents?

## Exercises

### Exercise 1 — Classify three addresses

**Goal:** Identify the region for each address you have already seen.

**Solution:**

```text
0x0000026C  Flash
0x200001E0  SRAM
0x400FE608  Peripheral register space
```

**Check:** Compare each address with the three ranges in the table.

### Exercise 2 — Calculate Flash capacity

**Goal:** Work out the size of the Flash range from its start and end addresses.

**Solution:**

```text
0x0003FFFF - 0x00000000 + 1 = 0x40000 bytes = 256 KiB
```

The `+ 1` counts both the first and last byte addresses.

**Check:** Your result should be `262,144` bytes.

### Exercise 3 — Calculate SRAM capacity

**Goal:** Work out the size of the SRAM range.

**Solution:**

```text
0x20007FFF - 0x20000000 + 1 = 0x8000 bytes = 32 KiB
```

**Check:** Your result should be `32,768` bytes.

### Exercise 4 — Find a variable's address

**Goal:** See that a runtime global variable is stored in SRAM.

**Solution:** In Watch, evaluate:

```c
&main_count
```

**Check:** The address should fall between `0x20000000` and `0x20007FFF`.

### Exercise 5 — Explain what survives power-off

**Goal:** Distinguish the firmware in Flash from a runtime variable in SRAM.

**Solution:** The program code at `0x0000026C` is in Flash and normally remains programmed when power is removed. `main_count` is runtime data in SRAM and should not be relied on after power is removed.

**Check:** Explain why the program can remain stored while the counter's runtime value does not.

## Next lesson

Next, we will look at how the linker and startup code place code and variables into these memory regions.
