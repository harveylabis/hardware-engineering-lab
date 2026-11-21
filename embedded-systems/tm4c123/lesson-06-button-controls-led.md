# Lesson 6 — Control the red LED with SW1

## Today's goal

Use the SW1 button to control the red LED.

- Press SW1: red LED on.
- Release SW1: red LED off.

SW1 is PF4. The red LED is PF1.

## Before you run

SW1 is active-low. Predict what value PF4 reads while SW1 is pressed. Should the red LED be on or off then?

## How it works

The program reads PF4 repeatedly in a loop. The pull-up resistor makes the input high when the button is released. Pressing the button connects PF4 to ground, so the input becomes low.

The program tests PF4:

```c
if (button_state == 0u)
```

Low means pressed. When pressed, the program writes a high value to PF1 to turn on the red LED.

## Register addresses

| Register | Address | Address calculation | What we do |
|---|---:|---|---|
| `SYSCTL_RCGCGPIO` | `0x400FE608` | `0x400FE000 + 0x608` | Enable the Port F clock |
| `SYSCTL_PRGPIO` | `0x400FEA08` | `0x400FE000 + 0xA08` | Wait until Port F is ready |
| `GPIO_PORTF_AFSEL` | `0x40025420` | `0x40025000 + 0x420` | Select regular GPIO for PF1 and PF4 |
| `GPIO_PORTF_DIR` | `0x40025400` | `0x40025000 + 0x400` | Make PF1 an output and PF4 an input |
| `GPIO_PORTF_PUR` | `0x40025510` | `0x40025000 + 0x510` | Enable the PF4 pull-up |
| `GPIO_PORTF_DEN` | `0x4002551C` | `0x40025000 + 0x51C` | Enable digital I/O for PF1 and PF4 |
| `GPIO_PORTF_AMSEL` | `0x40025528` | `0x40025000 + 0x528` | Disable analog mode on PF1 and PF4 |
| `GPIO_PORTF_DATA` for PF1 | `0x40025008` | `0x40025000 + (0x02 << 2)` | Write only the red LED pin |
| `GPIO_PORTF_DATA` for PF4 | `0x40025040` | `0x40025000 + (0x10 << 2)` | Read only the SW1 pin |

Port F APB base address is `0x40025000`. The GPIO register offsets are from the TM4C123GH6PM data sheet.

The `GPIODATA` addresses are pin-masked aliases:

```text
PF1 mask = 1 << 1 = 0x02
PF1 alias = 0x40025000 + (0x02 << 2) = 0x40025008

PF4 mask = 1 << 4 = 0x10
PF4 alias = 0x40025000 + (0x10 << 2) = 0x40025040
```

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

#define GPIO_PORTF_RED_DATA_R \
    (*((volatile uint32_t *)0x40025008u))
#define GPIO_PORTF_SW1_DATA_R \
    (*((volatile const uint32_t *)0x40025040u))

#define PORTF_CLOCK (1u << 5)
#define LED_RED     (1u << 1)
#define SW1         (1u << 4)

volatile uint32_t button_state;
volatile uint32_t led_output;

int main(void)
{
    SYSCTL_RCGCGPIO_R |= PORTF_CLOCK;

    while ((SYSCTL_PRGPIO_R & PORTF_CLOCK) == 0u)
    {
    }

    GPIO_PORTF_AFSEL_R &= ~(LED_RED | SW1);
    GPIO_PORTF_AMSEL_R &= ~(LED_RED | SW1);
    GPIO_PORTF_DIR_R |= LED_RED;
    GPIO_PORTF_DIR_R &= ~SW1;
    GPIO_PORTF_PUR_R |= SW1;
    GPIO_PORTF_DEN_R |= LED_RED | SW1;

    while (1)
    {
        button_state = GPIO_PORTF_SW1_DATA_R;

        if (button_state == 0u)
        {
            GPIO_PORTF_RED_DATA_R = LED_RED;
        }
        else
        {
            GPIO_PORTF_RED_DATA_R = 0u;
        }

        led_output = GPIO_PORTF_RED_DATA_R;
    }
}
```

The two `GPIODATA` aliases read and write different pins. Writing PF1 does not change PF4. Reading PF4 does not return the PF1 output.

## CCS steps

1. Open your `lesson01_did0` project in CCS.
2. Replace `main.c` with the code above and save it.
3. Choose **Project → Build Project**. Check the Console for errors.
4. Start **Debug As → Code Composer Debug Session**.
5. Add `button_state` and `led_output` to **Expressions**.
6. Run the program. Press and release SW1 while the program runs.
7. Check the red LED. It should light only while SW1 is pressed.

To inspect values in CCS, put a breakpoint on `led_output = GPIO_PORTF_RED_DATA_R;`. At that line, the current button has been read and the LED output has been written. Step over once to update `led_output`. To test the other button state, press or release SW1 while paused, then click **Resume**. At the next breakpoint, step over and inspect `button_state` and `led_output`.

## Expected result

| SW1 state | `button_state` | `led_output` | Red LED |
|---|---:|---:|---|
| Released | `0x10` | `0x00` | Off |
| Pressed | `0x00` | `0x02` | On |

## DriverLib comparison

The equivalent DriverLib setup and control use:

```c
SysCtlPeripheralEnable(SYSCTL_PERIPH_GPIOF);
while (!SysCtlPeripheralReady(SYSCTL_PERIPH_GPIOF))
{
}

GPIOPinTypeGPIOInput(GPIO_PORTF_BASE, GPIO_PIN_4);
GPIOPadConfigSet(GPIO_PORTF_BASE, GPIO_PIN_4,
                 GPIO_STRENGTH_2MA, GPIO_PIN_TYPE_STD_WPU);
GPIOPinTypeGPIOOutput(GPIO_PORTF_BASE, GPIO_PIN_1);

if (GPIOPinRead(GPIO_PORTF_BASE, GPIO_PIN_4) == 0u)
{
    GPIOPinWrite(GPIO_PORTF_BASE, GPIO_PIN_1, GPIO_PIN_1);
}
else
{
    GPIOPinWrite(GPIO_PORTF_BASE, GPIO_PIN_1, 0u);
}
```

The register-level example shows what the DriverLib calls configure.

## Your notes

- My prediction:
- Did the LED respond to SW1?:
- Released values for `button_state` and `led_output`:
- Pressed values for `button_state` and `led_output`:
- What is still unclear?:

## Short check

1. Why does SW1 read low when pressed?
2. What does the program write to PF1 when SW1 is pressed?
3. Which register makes PF1 an output?

## Exercises

### Exercise 1 — Make a pressed flag

**Goal:** Convert the raw PF4 reading into `1` when pressed and `0` when released.

**Solution:**

```c
volatile uint32_t sw1_pressed;

while (1)
{
    sw1_pressed =
        (GPIO_PORTF_SW1_DATA_R == 0u) ? 1u : 0u;

    GPIO_PORTF_RED_DATA_R =
        sw1_pressed ? LED_RED : 0u;
}
```

**Check:** Watch `sw1_pressed`. It should be `0` when released and `1` when pressed.

### Exercise 2 — Reverse the LED behavior

**Goal:** Turn the red LED on when SW1 is released and off when it is pressed.

**Solution:**

```c
while (1)
{
    button_state = GPIO_PORTF_SW1_DATA_R;
    GPIO_PORTF_RED_DATA_R =
        (button_state != 0u) ? LED_RED : 0u;
    led_output = GPIO_PORTF_RED_DATA_R;
}
```

**Check:** The red LED should be on when SW1 is released and off while pressed.

### Exercise 3 — Control the blue LED with SW1

**Goal:** Keep the same button behavior, but use PF2 and the blue LED.

**Solution:** Add this register alias and mask:

```c
#define GPIO_PORTF_BLUE_DATA_R \
    (*((volatile uint32_t *)0x40025010u))
#define LED_BLUE (1u << 2)
```

Add PF2 to the digital GPIO setup:

```c
GPIO_PORTF_AFSEL_R &= ~LED_BLUE;
GPIO_PORTF_AMSEL_R &= ~LED_BLUE;
GPIO_PORTF_DIR_R |= LED_BLUE;
GPIO_PORTF_DEN_R |= LED_BLUE;
```

Then use this loop:

```c
while (1)
{
    button_state = GPIO_PORTF_SW1_DATA_R;
    GPIO_PORTF_BLUE_DATA_R =
        (button_state == 0u) ? LED_BLUE : 0u;
}
```

**Check:** Pressing SW1 should turn on blue. The red LED should stay off.

### Exercise 4 — Toggle red once per press

**Goal:** Change the red LED state once when SW1 is pressed, rather than following the button level.

**Solution:** Add these variables:

```c
volatile uint32_t previous_pressed;
volatile uint32_t led_state;
```

Replace the loop with:

```c
while (1)
{
    uint32_t pressed =
        (GPIO_PORTF_SW1_DATA_R == 0u) ? 1u : 0u;

    if ((pressed != 0u) && (previous_pressed == 0u))
    {
        led_state ^= LED_RED;
        GPIO_PORTF_RED_DATA_R = led_state;
    }

    previous_pressed = pressed;
}
```

**Check:** Press and release SW1 repeatedly. The LED should change state on each press. A physical button can bounce, so it may occasionally toggle more than once; we will study debouncing later.

### Exercise 5 — Control green with SW2

**Goal:** Use SW2 on PF0 to control the green LED on PF3.

**Solution:** After enabling Port F's clock and waiting for it to be ready, unlock and commit PF0:

```c
#define GPIO_PORTF_LOCK_R (*((volatile uint32_t *)0x40025520u))
#define GPIO_PORTF_CR_R   (*((volatile uint32_t *)0x40025524u))
#define GPIO_PORTF_SW2_DATA_R \
    (*((volatile const uint32_t *)0x40025004u))
#define GPIO_PORTF_GREEN_DATA_R \
    (*((volatile uint32_t *)0x40025020u))

#define GPIO_LOCK_KEY 0x4C4F434Bu
#define SW2 (1u << 0)
#define LED_GREEN (1u << 3)

GPIO_PORTF_LOCK_R = GPIO_LOCK_KEY;
GPIO_PORTF_CR_R |= SW2;

GPIO_PORTF_AFSEL_R &= ~(SW2 | LED_GREEN);
GPIO_PORTF_AMSEL_R &= ~(SW2 | LED_GREEN);
GPIO_PORTF_DIR_R &= ~SW2;
GPIO_PORTF_DIR_R |= LED_GREEN;
GPIO_PORTF_PUR_R |= SW2;
GPIO_PORTF_DEN_R |= SW2 | LED_GREEN;
```

Use this loop:

```c
while (1)
{
    uint32_t sw2_pressed =
        (GPIO_PORTF_SW2_DATA_R == 0u) ? 1u : 0u;

    GPIO_PORTF_GREEN_DATA_R =
        sw2_pressed ? LED_GREEN : 0u;
}
```

**Check:** SW2 should control only the green LED. PF0 must be unlocked after Port F is ready and before changing its GPIO settings.

## Next lesson

Next, we will learn how GPIO pins can notify the CPU with interrupts instead of being checked continuously in a loop.
