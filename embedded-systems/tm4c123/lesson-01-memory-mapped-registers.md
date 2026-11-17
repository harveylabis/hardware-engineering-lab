# Lesson 1 — Reading a hardware register

## Today's goal

Read one TM4C123 register in C.

After this lesson, you should be able to explain:

- What a register address means.
- Why we use a pointer.
- Why the pointer is `volatile`.

## One idea

The microcontroller gives each hardware register an address.

The `DID0` register tells us about the chip. Its address is **`0x400FE000`**. It is a read-only, 32-bit register.

When C reads this address, the chip returns the register's value. This is called **memory-mapped I/O**.

## Look before you run

What do you think this code reads: a regular C variable, or the hardware at `0x400FE000`?

It is okay if you are not sure yet. Write down your guess, then check it as you follow the steps.

## The code

```c
#include <stdint.h>

#define SYSCTL_DID0_R (*((volatile const uint32_t *)0x400FE000u))

volatile uint32_t device_id;

int main(void)
{
    device_id = SYSCTL_DID0_R;

    while (1)
    {
    }
}
```

In plain language:

- `0x400FE000` is the register's address.
- The pointer tells C to look at that address.
- `*` means “read what is at this address.”
- `uint32_t` means the value is 32 bits.
- `volatile` tells the compiler to do the read instead of reusing an old value.
- `const` means this code only reads the register.
- `device_id` holds the value we read so we can inspect it.

This address is specific to the TM4C123. Do not use it on a different chip without checking that chip's manual.

## Hardware

No extra wires are needed. Connect the LaunchPad's ICDI USB connector to the Mac. macOS already sees the board's ICDI device.

## Open and set up CCS

1. Open **Code Composer Studio**.
2. Choose a workspace folder when asked. For example, create `ccs-workspaces/tm4c123` in your home folder. This keeps CCS settings outside the course files.
3. Choose **File → New → CCS Project**. If the welcome screen has a **Create a new project** option, that also opens the project wizard.
4. Enter a project name such as `lesson01_did0`.
5. Set the project location to this repository's `embedded-systems/tm4c123/lesson01_did0` folder.
6. Select device **TM4C123GH6PM**.
7. Select connection **Stellaris In-Circuit Debug Interface**. This is the LaunchPad's built-in ICDI debugger.
8. Select the **TI Arm Clang** compiler.
9. Select the empty C project template that creates `main.c`.
10. Click **Finish**.

CCS creates the startup and memory files that a runnable program needs. You do not need to write those files for this lesson.

If a choice is missing, stop and tell me what the screen shows. Do not choose a different device or debugger at random.

## Build and debug

1. Open the project's `main.c`.
2. Replace its contents with the code above, then save.
3. Select **Project → Build Project** (or click the hammer icon).
4. Check the **Console**. The build should finish without errors.
5. Right-click the project and choose **Debug As → Code Composer Debug Session**, or use the debug/bug button.
6. CCS switches to the debug view and loads the program onto the LaunchPad.
7. Click the line `device_id = SYSCTL_DID0_R;` to place a breakpoint there.
8. Click **Resume**. The program stops at that line.
9. Click **Step Over** once. The register is read.
10. Find `device_id` in the **Variables** view. Record its value.

The debugger should show the same value if you open **View → Memory Browser** and enter `0x400FE000`. If the menu names differ, send me a screenshot or the names you see.

## Expected result

The program reads `DID0` once, stores the result in `device_id`, then waits forever. It does not change any pins.

The value should match when viewed as `device_id` and at address `0x400FE000`. Record what you see; do not guess the value.

## Your notes

- Did the program build?:
- Did CCS connect to the LaunchPad?:
- What value did `device_id` show? **Observed: 402981122 (`0x18050102`)**
- Did the Memory Browser show the same value?:
- What is still unclear?:

## Short check

1. What does `0x400FE000` identify?
2. What does `volatile` ask the compiler to do?
3. Does this program change the LED?

## If something goes wrong

- **Build fails:** copy the first error from the Console and send it to me.
- **Debugger cannot connect:** check that the board is plugged into the ICDI USB connector. Then send me the CCS error.
- **Value is missing:** tell me which CCS view you are using. We can inspect it together.

## Next lesson

Next, we will practice pointers and bit operations before using them to control an LED.
