# Lesson 3 — Set and clear selected bits

## Today's goal

Change selected bits without changing the others.

Today we will work with a normal C variable in RAM. We will not write to a hardware register yet.

## One idea

Use bitwise OR (`|`) to set a bit to 1.

Use bitwise AND (`&`) with an inverted mask (`~`) to clear a bit to 0.

Start with this value:

```text
state = 0x01 = 0000 0001
```

Set bit 2:

```text
mask       = 0000 0100
state OR mask = 0000 0101 = 0x05
```

Then clear bit 0:

```text
mask       = 0000 0001
~mask      = 1111 1110
state AND ~mask = 0000 0100 = 0x04
```

Bit numbers start at zero. Bit 0 is the rightmost bit.

## Before you run

Predict the value after each operation:

1. What is `state` after setting bit 2?
2. What is `state` after clearing bit 0?

## The code

Replace the contents of `main.c` with this:

```c
#include <stdint.h>

volatile uint32_t state = 0x01u;

int main(void)
{
    state |= (1u << 2);
    state &= ~(1u << 0);

    while (1)
    {
    }
}
```

Read the statements from the inside out:

- `1u << 2` makes the mask `0x04`.
- `state |= mask` sets the mask's bit in `state`.
- `1u << 0` makes the mask `0x01`.
- `~mask` flips all bits in the mask.
- `state &= ~mask` clears that bit in `state`.

The other bits stay the same.

## CCS steps

1. Open your `lesson01_did0` project.
2. Open `main.c`, replace its contents with the code above, and save.
3. Choose **Project → Build Project**. Check the Console for errors.
4. Start **Debug As → Code Composer Debug Session**.
5. Add a breakpoint on `state |= (1u << 2);`.
6. Click **Resume**. Inspect `state` in **Variables** or **Expressions**. It should be `0x01`.
7. Click **Step Over** once. Inspect `state` again. It should be `0x05`.
8. Click **Step Over** again. The value should be `0x04`.

CCS may show decimal values. For example, `0x05` is decimal `5`.

## Expected result

| Point in the program | `state` |
|---|---:|
| Before the first statement | `0x01` |
| After setting bit 2 | `0x05` |
| After clearing bit 0 | `0x04` |

## Your notes

- My first prediction was `3`, then `2`. I counted bit positions differently.
- Remember: C numbers bits from zero. Bit 0 has value `0x01`; bit 2 has value `0x04`.
- CCS values observed: `0x05` after setting bit 2, then `0x04` after clearing bit 0.
- Which step is still unclear?:

## Short check

1. Which operator sets a selected bit?
2. Which expression clears a selected bit?
3. Why does clearing bit 0 leave bit 2 unchanged?

## Exercises

### Exercise 1 — Set a bit

**Goal:** Set bit 3 in `state = 0x01`.

**Solution:**

```c
uint32_t state = 0x01u;
state |= (1u << 3);
```

The result is `0x09`.

### Exercise 2 — Clear a bit

**Goal:** Clear bit 2 from `state = 0x05`.

**Solution:**

```c
uint32_t state = 0x05u;
state &= ~(1u << 2);
```

The result is `0x01`.

### Exercise 3 — Toggle a bit

**Goal:** Toggle bit 1 in `state = 0x05`.

**Solution:**

```c
uint32_t state = 0x05u;
state ^= (1u << 1);
```

The result is `0x07`.

### Exercise 4 — Set two bits

**Goal:** Set bits 1 and 3 in `state = 0x00`.

**Solution:**

```c
uint32_t state = 0x00u;
state |= (1u << 1) | (1u << 3);
```

The result is `0x0A`.

### Exercise 5 — Preserve unselected bits

**Goal:** Clear bit 0 while preserving all other bits in `state = 0xA5`.

**Solution:**

```c
uint32_t state = 0xA5u;
state &= ~(1u << 0);
```

The result is `0xA4`.

## Next lesson

Next, we will use these operations on a GPIO register to control the LaunchPad's RGB LED in [Lesson 4](lesson-04-gpio-red-led.md).
