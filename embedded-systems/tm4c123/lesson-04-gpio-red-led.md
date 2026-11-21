# Lesson 4 — Turn on the red LED

## Today's goal

Use GPIO registers to turn on the LaunchPad's red LED.

The red LED is connected to pin **PF1**. We will set PF1 high.

## Before you run

What do you expect the LED to do when the program writes a `1` to PF1?

## The registers

The CPU cannot use Port F until we enable its clock. Then we configure PF1 as a digital output.

| Register | Address | Address calculation | What we do |
|---|---:|---|---|
| `SYSCTL_RCGCGPIO` | `0x400FE608` | `0x400FE000 + 0x608` | Set bit 5 to enable Port F's clock |
| `SYSCTL_PRGPIO` | `0x400FEA08` | `0x400FE000 + 0xA08` | Wait until bit 5 says Port F is ready |
| `GPIO_PORTF_AFSEL` | `0x40025420` | `0x40025000 + 0x420` | Clear bit 1 to select regular GPIO |
| `GPIO_PORTF_AMSEL` | `0x40025528` | `0x40025000 + 0x528` | Clear bit 1 to disable analog mode |
| `GPIO_PORTF_DIR` | `0x40025400` | `0x40025000 + 0x400` | Set bit 1 to make PF1 an output |
| `GPIO_PORTF_DEN` | `0x4002551C` | `0x40025000 + 0x51C` | Set bit 1 to enable digital I/O |
| `GPIO_PORTF_DATA` | `0x40025008` | `0x40025000 + (0x02 << 2)` | Write `0x02` to set PF1 high |

For these Port F registers, bit 1 is PF1. In the clock registers, bit 5 selects Port F.

The DATA address ends in `008` so this access selects only PF1. That lets us change the red LED without changing the other Port F pins.

## How these addresses were obtained

The TM4C123GH6PM datasheet has these bases and offsets:

```text
System Control base = 0x400FE000
Port F APB base     = 0x40025000
```

An address is:

```text
base address + register offset
```

Examples:

```text
RCGCGPIO:
0x400FE000 + 0x608 = 0x400FE608

PRGPIO:
0x400FE000 + 0xA08 = 0x400FEA08

GPIODIR:
0x40025000 + 0x400 = 0x40025400

GPIODEN:
0x40025000 + 0x51C = 0x4002551C

GPIOAMSEL:
0x40025000 + 0x528 = 0x40025528

GPIOAFSEL:
0x40025000 + 0x420 = 0x40025420
```

`GPIODATA` is different. It has aliases for selected pins:

```text
alias address = Port F base + (pin mask << 2)

PF1 pin mask = 0x02
0x02 << 2 = 0x08
0x40025000 + 0x08 = 0x40025008
```

The shift by 2 is required because `GPIODATA` address lines `A9:2` represent GPIO bits 7:0. GPIO bit 1 becomes address bit `1 + 2 = 3`, whose hex value is `0x08`. Accessing `0x40025008` selects PF1 for that read or write.

## The code

Replace the contents of `main.c` with this:

```c
#include <stdint.h>

#define SYSCTL_RCGCGPIO_R  (*((volatile uint32_t *)0x400FE608u))
#define SYSCTL_PRGPIO_R    (*((volatile const uint32_t *)0x400FEA08u))

#define GPIO_PORTF_AFSEL_R (*((volatile uint32_t *)0x40025420u))
#define GPIO_PORTF_DIR_R   (*((volatile uint32_t *)0x40025400u))
#define GPIO_PORTF_DEN_R   (*((volatile uint32_t *)0x4002551Cu))
#define GPIO_PORTF_AMSEL_R (*((volatile uint32_t *)0x40025528u))
#define GPIO_PORTF_DATA_R  (*((volatile uint32_t *)0x40025008u))

#define PORTF_CLOCK  (1u << 5)
#define LED_RED      (1u << 1)

volatile uint32_t led_output;

int main(void)
{
    SYSCTL_RCGCGPIO_R |= PORTF_CLOCK;

    while ((SYSCTL_PRGPIO_R & PORTF_CLOCK) == 0u)
    {
    }

    GPIO_PORTF_AFSEL_R &= ~LED_RED;
    GPIO_PORTF_AMSEL_R &= ~LED_RED;
    GPIO_PORTF_DIR_R |= LED_RED;
    GPIO_PORTF_DEN_R |= LED_RED;

    GPIO_PORTF_DATA_R = LED_RED;
    led_output = GPIO_PORTF_DATA_R;

    while (1)
    {
    }
}
```

The code does these steps:

1. Enable the Port F clock.
2. Wait until Port F is ready.
3. Configure PF1 as a digital output.
4. Write `1` to PF1. The red LED turns on.

`led_output` saves the value we read back. CCS can show this variable in the Watch view.

## CCS steps

1. Connect the LaunchPad to the Mac.
2. Open your `lesson01_did0` project and its `main.c`.
3. Replace the code with the example above. Save the file.
4. Choose **Project → Build Project**. Check the Console for errors.
5. Start **Debug As → Code Composer Debug Session**.
6. Put a breakpoint on `GPIO_PORTF_DATA_R = LED_RED;`.
7. Click **Resume**, then **Step Over** once.
8. Look at the LaunchPad. The red LED should be on.
9. Click **Step Over** once more. This reads PF1 into `led_output`.
10. In **Variables** or **Expressions**, inspect `led_output`. It should be `0x02`.

If the LED is off, check the board is powered and the program reached the write to `GPIO_PORTF_DATA_R`.

## Expected result

- The LaunchPad's red RGB LED turns on.
- `led_output` reads `0x02`.
- The blue and green LEDs stay off.

## DriverLib comparison

DriverLib provides functions that perform the same setup:

```c
SysCtlPeripheralEnable(SYSCTL_PERIPH_GPIOF);
while (!SysCtlPeripheralReady(SYSCTL_PERIPH_GPIOF))
{
}

GPIOPinTypeGPIOOutput(GPIO_PORTF_BASE, GPIO_PIN_1);
GPIOPinWrite(GPIO_PORTF_BASE, GPIO_PIN_1, GPIO_PIN_1);
```

For now, use the register-level code above. The DriverLib calls are a shorter way to request the same setup.

## Your notes

- My prediction: PF1 goes high and switches on the red LED through its driver transistor.
- Did the red LED turn on?: **Yes**
- What value did `led_output` show?: **Observed: `0x02` (decimal 2)**
- What is still unclear?:

## Short check

1. Why does the code enable the Port F clock first? **Because it's needed to run GPIO**
2. Which bit controls PF1? **bit 1**
3. Which register makes PF1 an output? **GPIODIR**

## Exercises

### Exercise 1 — Calculate the PF1 address

**Goal:** Calculate the PF1 masked `GPIODATA` address.

**Solution:**

```text
PF1 pin mask = 0x02
0x02 << 2 = 0x08
0x40025000 + 0x08 = 0x40025008
```

### Exercise 2 — Turn on blue

**Goal:** Configure PF2 and turn on the blue LED.

**Solution:**

```c
#define LED_BLUE (1u << 2)

GPIO_PORTF_AFSEL_R &= ~LED_BLUE;
GPIO_PORTF_AMSEL_R &= ~LED_BLUE;
GPIO_PORTF_DIR_R |= LED_BLUE;
GPIO_PORTF_DEN_R |= LED_BLUE;
```

Use `GPIODATA` alias `0x40025010` to write PF2.

### Exercise 3 — Turn on green

**Goal:** Configure PF3 and turn on the green LED.

**Solution:**

```c
#define LED_GREEN (1u << 3)

GPIO_PORTF_AFSEL_R &= ~LED_GREEN;
GPIO_PORTF_AMSEL_R &= ~LED_GREEN;
GPIO_PORTF_DIR_R |= LED_GREEN;
GPIO_PORTF_DEN_R |= LED_GREEN;
```

Use `GPIODATA` alias `0x40025020` to write PF3.

### Exercise 4 — Turn off red

**Goal:** Clear PF1 without changing PF2 or PF3.

**Solution:**

```c
#define GPIO_PORTF_RED_DATA_R (*((volatile uint32_t *)0x40025008u))
#define LED_RED (1u << 1)

GPIO_PORTF_RED_DATA_R &= ~LED_RED;
```

### Exercise 5 — Turn on red and blue

**Goal:** Set PF1 and PF2 with one alias.

**Solution:**

```c
#define GPIO_PORTF_RED_BLUE_DATA_R (*((volatile uint32_t *)0x40025018u))
#define LED_RED_BLUE ((1u << 1) | (1u << 2))

GPIO_PORTF_RED_BLUE_DATA_R = LED_RED_BLUE;
```

### Exercise 6 — Turn on all RGB colors

**Goal:** Write red, green, and blue through one combined alias.

**Solution:**

```text
LED_RED   = 0x02
LED_BLUE  = 0x04
LED_GREEN = 0x08
Combined mask = 0x0E
0x0E << 2 = 0x38
0x40025000 + 0x38 = 0x40025038
```

```c
#define GPIO_PORTF_RGB_DATA_R \
    (*((volatile uint32_t *)0x40025038u))

#define LED_MASK ((1u << 1) | (1u << 2) | (1u << 3))

GPIO_PORTF_RGB_DATA_R = LED_MASK;
```

### Exercise 7 — Change only green

**Goal:** Set, clear, or toggle PF3 without changing PF1 or PF2.

**Solution:**

```c
#define LED_GREEN (1u << 3)
#define GPIO_PORTF_RGB_DATA_R \
    (*((volatile uint32_t *)0x40025038u))

/* Set */
GPIO_PORTF_RGB_DATA_R |= LED_GREEN;

/* Clear */
GPIO_PORTF_RGB_DATA_R &= ~LED_GREEN;

/* Toggle */
GPIO_PORTF_RGB_DATA_R ^= LED_GREEN;
```

## Next lesson

Next, we will turn the LED off and learn how to read a button.
