# Lesson 2 — Keep part of a value

## Today's goal

Use a bit mask to keep the lowest 8 bits of a number.

## One idea

The value you read last lesson was:

```text
device_id = 0x18050102
```

In C, `&` is the bitwise AND operator. It compares each pair of bits:

| First bit | Second bit | Result |
|---:|---:|---:|
| 0 | either value | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 1 |

The mask `0xFF` has eight `1` bits. AND-ing a value with `0xFF` keeps its lowest 8 bits and clears the rest:

```text
0x18050102
& 0x000000FF
------------
  0x00000002
```

## Before you run

What value do you expect `low_byte` to contain?

## The code

Replace the contents of `main.c` with this:

```c
#include <stdint.h>

#define SYSCTL_DID0_R (*((volatile const uint32_t *)0x400FE000u))

volatile uint32_t device_id;
volatile uint32_t low_byte;

int main(void)
{
    device_id = SYSCTL_DID0_R;
    low_byte = device_id & 0xFFu;

    while (1)
    {
    }
}
```

`0xFFu` is the mask. The `u` means the number is unsigned.

This code only reads the chip ID. It does not change any hardware.

## CCS steps

1. Open your `lesson01_did0` project in CCS.
2. Open `main.c` and replace its contents with the code above.
3. Save the file.
4. Choose **Project → Build Project**. Check the Console for build errors.
5. Start a debug session with **Debug As → Code Composer Debug Session**.
6. Put a breakpoint on `low_byte = device_id & 0xFFu;`.
7. Click **Resume**, then **Step Over** once.
8. In **Variables** or **Expressions**, inspect `device_id` and `low_byte`.

If `device_id` or `low_byte` is not shown, make sure the selected stack frame is `main`.

## Expected result

If `device_id` is still `0x18050102`, then `low_byte` should be:

```text
0x00000002
```

CCS may display the value in decimal or hexadecimal.

For the TM4C123GH6PM, the data sheet names DID0's lowest 8 bits `MINOR`. So `low_byte` is the minor die revision. The value `0x02` means the second metal-layer change. You correctly found this in the data sheet.

## Your notes

- What did you predict?: **2**
- What value did `low_byte` show? **Observed: `2`**
- Did CCS show decimal or hexadecimal? **decimal**

## Short check

1. Which C operator applies the mask?
2. How many bits does `0xFF` keep?
3. Did this code write to the `DID0` register?

## Next lesson

Next, we will use a mask to change selected bits in a value. Then we will use the same idea to control an LED.
