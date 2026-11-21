# Lesson 5 — Read the SW1 button

## Today's goal

Read the LaunchPad **SW1** button from GPIO.

SW1 is connected to **PF4**.

## One idea

Set PF4 as a digital input.

Enable its pull-up resistor. This keeps PF4 high when the button is not pressed.

When SW1 is pressed, PF4 connects to ground. The PF4 input value becomes low.

| SW1 state | PF4 value |
|---|---:|
| Released | `1` |
| Pressed | `0` |

## Before you run

Predict:

1. What should you read when SW1 is released?
2. What should you read when SW1 is pressed?

## The registers

| Register | Address | Address calculation | What we do |
|---|---:|---|---|
| `SYSCTL_RCGCGPIO` | `0x400FE608` | `0x400FE000 + 0x608` | Set bit 5 to enable Port F |
| `SYSCTL_PRGPIO` | `0x400FEA08` | `0x400FE000 + 0xA08` | Wait until Port F is ready |
| `GPIO_PORTF_LOCK` | `0x40025520` | `0x40025000 + 0x520` | Unlock protected PF0 before using SW2 |
| `GPIO_PORTF_CR` | `0x40025524` | `0x40025000 + 0x524` | Commit PF0 so its GPIO settings can be changed |
| `GPIO_PORTF_AFSEL` | `0x40025420` | `0x40025000 + 0x420` | Clear PF4 to select regular GPIO |
| `GPIO_PORTF_DIR` | `0x40025400` | `0x40025000 + 0x400` | Clear PF4 to make PF4 an input |
| `GPIO_PORTF_PUR` | `0x40025510` | `0x40025000 + 0x510` | Set PF4 to enable its pull-up |
| `GPIO_PORTF_DEN` | `0x4002551C` | `0x40025000 + 0x51C` | Set PF4 to enable digital input |
| `GPIO_PORTF_AMSEL` | `0x40025528` | `0x40025000 + 0x528` | Clear PF4 to disable analog mode |
| `GPIO_PORTF_DATA` | `0x40025040` | `0x40025000 + (0x10 << 2)` | Read PF4 through a PF4-only address |
| `GPIO_PORTF_DATA` | `0x40025004` | `0x40025000 + (0x01 << 2)` | Read PF0 (SW2) through a PF0-only address |

PF4 is bit 4. The DATA address `0x40025040` uses the Port F base plus the PF4 address mask.

SW2 uses PF0. PF0 is protected because it can also be the NMI input. Before configuring PF0, enable and wait for the Port F clock, then unlock and commit PF0. Do not access the Port F registers before the peripheral is ready.

## How these addresses were obtained

Use the Port F APB base and each register offset:

```text
Port F APB base = 0x40025000
Port F AFSEL offset = 0x420
0x40025000 + 0x420 = 0x40025420

GPIODIR offset = 0x400
0x40025000 + 0x400 = 0x40025400

GPIOPUR offset = 0x510
0x40025000 + 0x510 = 0x40025510

GPIODEN offset = 0x51C
0x40025000 + 0x51C = 0x4002551C

GPIOAMSEL offset = 0x528
0x40025000 + 0x528 = 0x40025528

GPIOLOCK offset = 0x520
0x40025000 + 0x520 = 0x40025520

GPIOCR offset = 0x524
0x40025000 + 0x524 = 0x40025524
```

The clock addresses use the System Control base:

```text
System Control base = 0x400FE000
RCGCGPIO offset = 0x608
0x400FE000 + 0x608 = 0x400FE608

PRGPIO offset = 0xA08
0x400FE000 + 0xA08 = 0x400FEA08
```

The PF4 `GPIODATA` alias is calculated from its pin mask:

```text
PF4 pin mask = 0x10
0x10 << 2 = 0x40
0x40025000 + 0x40 = 0x40025040
```

### Why is the pin mask shifted by 2?

PF4 is GPIO data bit 4:

```text
PF4 pin mask = 1 << 4 = 0x10
```

But address bits `A9:2` correspond to GPIO data bits 7:0:

```text
address bit 2 corresponds to GPIO data bit 0
address bit 3 corresponds to GPIO data bit 1
address bit 4 corresponds to GPIO data bit 2
...
address bit 6 corresponds to GPIO data bit 4
```

Therefore, GPIO data bit 4 must move from position 4 to address position `4 + 2 = 6`:

```text
0x10 << 2 = 0x40
```

The two extra zeros in the shift are why address line `A6`, and not address line `A4`, selects PF4.

The datasheet uses the phrase “eight address lines [9:2].” This means:

```text
address A9:2 = data pin mask
```

Examples:

```text
0x40025040 has address mask 0x10
This selects GPIO data bit 4, which is PF4.

0x40025008 has address mask 0x02
This selects GPIO data bit 1, which is PF1.
```

For reads, address `0x40025040` returns PF4 and masks the other Port F input values out.

For writes, address `0x40025040` controls only PF4. The other Port F output pins keep their previous values. We use this alias when writing PF1 or reading PF4.

## The code

Replace `main.c` with this code:

```c
#include <stdint.h>

#define SYSCTL_RCGCGPIO_R  (*((volatile uint32_t *)0x400FE608u))
#define SYSCTL_PRGPIO_R    (*((volatile const uint32_t *)0x400FEA08u))

#define GPIO_PORTF_AFSEL_R (*((volatile uint32_t *)0x40025420u))
#define GPIO_PORTF_DIR_R   (*((volatile uint32_t *)0x40025400u))
#define GPIO_PORTF_PUR_R   (*((volatile uint32_t *)0x40025510u))
#define GPIO_PORTF_DEN_R   (*((volatile uint32_t *)0x4002551Cu))
#define GPIO_PORTF_AMSEL_R (*((volatile uint32_t *)0x40025528u))
#define GPIO_PORTF_DATA_R  (*((volatile const uint32_t *)0x40025040u))

#define PORTF_CLOCK (1u << 5)
#define SW1         (1u << 4)

volatile uint32_t button_state;

int main(void)
{
    SYSCTL_RCGCGPIO_R |= PORTF_CLOCK;

    while ((SYSCTL_PRGPIO_R & PORTF_CLOCK) == 0u)
    {
    }

    GPIO_PORTF_AFSEL_R &= ~SW1;
    GPIO_PORTF_AMSEL_R &= ~SW1;
    GPIO_PORTF_DIR_R &= ~SW1;
    GPIO_PORTF_PUR_R |= SW1;
    GPIO_PORTF_DEN_R |= SW1;

    while (1)
    {
        button_state = GPIO_PORTF_DATA_R & SW1;
    }
}
```

The assignment stores `0x10` when SW1 is released and `0x00` when it is pressed.

This form makes it easy to inspect the PF4 bit. The raw button state is:

```text
Released: 0x10
Pressed:  0x00
```

## CCS steps

1. Open `lesson01_did0` and replace `main.c` with the code above.
2. Save and choose **Project → Build Project**.
3. Start **Debug As → Code Composer Debug Session**.
4. Add a breakpoint on `button_state = GPIO_PORTF_DATA_R & SW1;`.
5. Click **Resume**.
6. CCS pauses at the assignment every time the loop executes.
7. Do not press SW1. Step over the assignment and inspect `button_state`.
8. Resume, then press and hold SW1.
9. Click **Pause** and inspect `button_state`.
10. Release SW1 and inspect `button_state` again.

The loop runs quickly, so pausing at an arbitrary time is fine. Resume and pause while SW1 is held.

## Expected result

- SW1 released: `button_state = 0x10`.
- SW1 pressed: `button_state = 0x00`.

## DriverLib comparison

The equivalent setup and read are:

```c
SysCtlPeripheralEnable(SYSCTL_PERIPH_GPIOF);
while (!SysCtlPeripheralReady(SYSCTL_PERIPH_GPIOF))
{
}

GPIOPinTypeGPIOInput(GPIO_PORTF_BASE, GPIO_PIN_4);
GPIOPadConfigSet(GPIO_PORTF_BASE, GPIO_PIN_4,
                 GPIO_STRENGTH_2MA, GPIO_PIN_TYPE_STD_WPU);

uint32_t value = GPIOPinRead(GPIO_PORTF_BASE, GPIO_PIN_4);
```

The register-level code shows what these DriverLib calls configure.

## Your notes

- My prediction:
- Released value:
- Pressed value:
- Did the values match the pull-up behavior?:

## Short check

1. Which pin is SW1? **PF4**
2. Is PF4 an input or an output? **input**
3. Why is the released value `0x10` instead of `1`? **Because button_state is a Port F data-mask result, not logical 0/1**

## Exercises

### Exercise 1 — Convert SW1 to a Boolean

**Goal:** Create variables that contain `1` for released and `0` for pressed.

**Solution:**

```c
#define SW1 (1u << 4)
#define GPIO_PORTF_SW1_DATA_R \
    (*((volatile uint32_t *)0x40025040u))

uint32_t sw1_pressed =
    ((GPIO_PORTF_SW1_DATA_R & SW1) == 0u) ? 1u : 0u;
```

This is an extension: first read the hardware mask, then change it into application logic.

### Exercise 2 — Read each button's state

**Goal:** Read both SW1 and SW2 without changing them to a value in GPIO bit position.

**Solution:**

After the Port F clock-ready loop, unlock and commit PF0 before configuring it:

```c
#define SW1 (1u << 4)
#define SW2 (1u << 0)
#define GPIO_LOCK_KEY 0x4C4F434Bu

#define GPIO_PORTF_LOCK_R (*((volatile uint32_t *)0x40025520u))
#define GPIO_PORTF_CR_R   (*((volatile uint32_t *)0x40025524u))

GPIO_PORTF_LOCK_R = GPIO_LOCK_KEY;
GPIO_PORTF_CR_R |= SW2;

GPIO_PORTF_AFSEL_R &= ~SW2;
GPIO_PORTF_AMSEL_R &= ~SW2;
GPIO_PORTF_DIR_R &= ~SW2;
GPIO_PORTF_PUR_R |= SW2;
GPIO_PORTF_DEN_R |= SW2;
```

Now define the masked data addresses and read the buttons:

```c

#define GPIO_PORTF_SW1_DATA_R \
    (*((volatile uint32_t *)0x40025040u))

#define GPIO_PORTF_SW2_DATA_R \
    (*((volatile uint32_t *)0x40025004u))

uint32_t sw1_pressed =
    ((GPIO_PORTF_SW1_DATA_R & SW1) == 0u) ? 1u : 0u;

uint32_t sw2_pressed =
    ((GPIO_PORTF_SW2_DATA_R & SW2) == 0u) ? 1u : 0u;
```

The unlock must happen after Port F's peripheral-ready check. Otherwise, a register access can cause a BusFault and enter `FaultISR`.

Expected combinations:

| SW1 | SW2 | `sw1_pressed` | `sw2_pressed` |
|---|---|---:|---:|
| Released | Released | 0 | 0 |
| Pressed | Released | 1 | 0 |
| Released | Pressed | 0 | 1 |
| Both pressed | Both pressed | 1 | 1 |

### Exercise 3 — Detect whether both buttons are pressed

**Goal:** Use one condition to detect SW1 and SW2 together.

**Solution:**

```c
if ((((GPIO_PORTF_SW1_DATA_R & SW1) == 0u) &&
     ((GPIO_PORTF_SW2_DATA_R & SW2) == 0u)))
{
    /* Both buttons are pressed */
}
```

Notice that this is a read-only GPIO exercise. It does not control the RGB LED.

### Exercise 4 — Detect a button position change

**Goal:** Record the previous PF4 state and print or inspect only changes.

**Solution:**

```c
uint32_t previous = 1u;

while (1)
{
    uint32_t current =
        ((GPIO_PORTF_SW1_DATA_R & SW1) != 0u) ? 1u : 0u;

    if (current != previous)
    {
        button_state = current;
        previous = current;
    }
}
```

`1` means released and `0` means pressed.

### Exercise 5 — Detect an edge using a bit mask

**Goal:** Keep separate masks for the previous and current states, then detect any changed button.

**Solution:**

```c
#define SW1 (1u << 4)
#define SW2 (1u << 0)

uint32_t previous = 0u;

while (1)
{
    uint32_t current =
        ((GPIO_PORTF_SW1_DATA_R & SW1) == 0u ? 0x10u : 0u) |
        ((GPIO_PORTF_SW2_DATA_R & SW2) == 0u ? 0x01u : 0u);

    uint32_t changed = current ^ previous;

    if (changed != 0u)
    {
        button_state = changed;
        previous = current;
    }
}
```

This combines both button masks into a small state variable. Bit 4 represents SW1 and bit 0 represents SW2.

The LaunchPad buttons are **active-low**:

| Button | Raw GPIO value | Ternary result |
|---|---:|---:|
| SW1 released | `0x10` | `0x00` |
| SW1 pressed | `0x00` | `0x10` |
| SW2 released | `0x01` | `0x00` |
| SW2 pressed | `0x00` | `0x01` |

So the ternary does more than copy the `GPIODATA` value. It changes the sense of the signal: a pressed button becomes a set bit in `current`. The `0x10` and `0x01` results are the same bit masks as PF4 and PF0, but they are returned when the raw pin reads low.

You could combine the raw readings directly:

```c
uint32_t raw_buttons =
    (GPIO_PORTF_SW1_DATA_R & SW1) |
    (GPIO_PORTF_SW2_DATA_R & SW2);
```

Yes, you can compare that raw snapshot with the previous one. Initialize `previous` after configuring the pins so the first loop does not report a fake change:

```c
uint32_t previous =
    GPIO_PORTF_SW1_DATA_R | GPIO_PORTF_SW2_DATA_R;

while (1)
{
    uint32_t current =
        GPIO_PORTF_SW1_DATA_R | GPIO_PORTF_SW2_DATA_R;

    uint32_t changed = current ^ previous;

    if (changed != 0u)
    {
        button_state = changed;
        previous = current;
    }
}
```

`current != previous` would also tell you whether any button changed. XOR gives you a mask identifying which bit changed:

```text
changed & SW1 is nonzero -> SW1 changed
changed & SW2 is nonzero -> SW2 changed
```

With raw values, a `1` means released and a `0` means pressed. For example, both released is `0x11`; pressing SW1 changes the snapshot to `0x01`.

The program must test the `changed` mask to name the button:

```c
if ((changed & SW1) != 0u)
{
    /* SW1 changed */
}

if ((changed & SW2) != 0u)
{
    /* SW2 changed */
}
```

Example: if `changed` is `0x10`, then `changed & SW1` is nonzero and `changed & SW2` is zero. That tells us PF4/SW1 changed. If `changed` is `0x01`, PF0/SW2 changed. The `button_state = changed;` assignment only saves the mask; it does not name the button by itself.

If you want a pressed-button mask instead, invert each masked input:

```c
uint32_t current =
    (~GPIO_PORTF_SW1_DATA_R & SW1) |
    (~GPIO_PORTF_SW2_DATA_R & SW2);
```

The `& SW1` and `& SW2` masks keep only the relevant pin bits after inversion. For a beginner, the ternary version makes the active-low conversion easier to see.

### Short challenges

Complete these without running the code:

1. What address reads only PF4?
2. What address reads only PF0?
3. What is the released and pressed result for PF4?
4. Write a Boolean expression that is `1` only when SW1 is pressed.
5. Write a mask expression that detects either SW1 or SW2 being pressed.

## Next lesson

Next, we will control the red LED with SW1 or move to GPIO interrupts.
