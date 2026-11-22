# Lesson 7 — Respond to a GPIO interrupt

## Today's goal

Use SW1 to generate an interrupt that toggles the red LED.

- Press SW1: the red LED changes state once.
- Release SW1: no action.

SW1 is PF4. The red LED is PF1.

## Before you run

In Lesson 6, the program repeatedly checked PF4 in a loop. This is called **polling**.

This time, the GPIO hardware will notify the CPU when PF4 changes from high to low. The CPU pauses its normal work and runs an **interrupt service routine (ISR)**.

Predict: Will the ISR run while SW1 is held down, or when SW1 is first pressed?

## How it works

SW1 is active-low. Its signal changes from high to low when pressed. We configure PF4 to detect that falling edge.

The interrupt path has three parts:

1. Port F detects the PF4 edge and raises its interrupt.
2. The NVIC allows the Port F interrupt to reach the Cortex-M4.
3. The CPU runs `GPIOPortFIntHandler`, which clears the interrupt and toggles the LED.

`led_state` is a software copy of the LED output. XOR flips only the bit selected by `LED_RED`:

```c
led_state ^= LED_RED;
GPIO_PORTF_RED_DATA_R = led_state;
```

The ISR must clear the GPIO interrupt flag. `GPIOICR` is write-one-to-clear: writing `SW1` clears PF4's flag. Do not use `|=` with this register.

## Register addresses

| Register | Address | Address calculation | What we do |
|---|---:|---|---|
| `SYSCTL_RCGCGPIO` | `0x400FE608` | `0x400FE000 + 0x608` | Enable the Port F clock |
| `SYSCTL_PRGPIO` | `0x400FEA08` | `0x400FE000 + 0xA08` | Wait until Port F is ready |
| `GPIO_PORTF_IS` | `0x40025404` | `0x40025000 + 0x404` | Select edge or level detection |
| `GPIO_PORTF_IBE` | `0x40025408` | `0x40025000 + 0x408` | Select one edge or both edges |
| `GPIO_PORTF_IEV` | `0x4002540C` | `0x40025000 + 0x40C` | Select rising or falling edge |
| `GPIO_PORTF_IM` | `0x40025410` | `0x40025000 + 0x410` | Unmask the PF4 interrupt |
| `GPIO_PORTF_RIS` | `0x40025414` | `0x40025000 + 0x414` | Check whether PF4 detected an interrupt condition |
| `GPIO_PORTF_MIS` | `0x40025418` | `0x40025000 + 0x418` | Check whether PF4 caused the interrupt |
| `GPIO_PORTF_ICR` | `0x4002541C` | `0x40025000 + 0x41C` | Clear the PF4 interrupt flag |
| `NVIC_EN0` | `0xE000E100` | NVIC base `0xE000E000` + `0x100` | Enable Port F IRQ 30 |
| `NVIC_UNPEND0` | `0xE000E280` | NVIC base `0xE000E000` + `0x280` | Clear an old pending Port F interrupt |
| `GPIO_PORTF_DATA` for PF1 | `0x40025008` | `0x40025000 + (0x02 << 2)` | Write the red LED pin |
| `GPIO_PORTF_DATA` for PF4 | `0x40025040` | `0x40025000 + (0x10 << 2)` | Read the SW1 pin |

Port F's interrupt number is 30. It uses bit 30 in the first NVIC enable register.

For `GPIOIS`, `GPIOIBE`, and `GPIOIEV`, a zero in PF4's bit selects edge detection, a single edge, and a falling edge, respectively.

## The code

Replace `main.c` with this code:

```c
#include <stdint.h>

#define SYSCTL_RCGCGPIO_R  (*((volatile uint32_t *)0x400FE608u))
#define SYSCTL_PRGPIO_R    (*((volatile const uint32_t *)0x400FEA08u))

#define GPIO_PORTF_IS_R    (*((volatile uint32_t *)0x40025404u))
#define GPIO_PORTF_IBE_R   (*((volatile uint32_t *)0x40025408u))
#define GPIO_PORTF_IEV_R   (*((volatile uint32_t *)0x4002540Cu))
#define GPIO_PORTF_IM_R    (*((volatile uint32_t *)0x40025410u))
#define GPIO_PORTF_MIS_R   (*((volatile const uint32_t *)0x40025418u))
#define GPIO_PORTF_ICR_R   (*((volatile uint32_t *)0x4002541Cu))

#define GPIO_PORTF_AFSEL_R (*((volatile uint32_t *)0x40025420u))
#define GPIO_PORTF_DIR_R   (*((volatile uint32_t *)0x40025400u))
#define GPIO_PORTF_PUR_R   (*((volatile uint32_t *)0x40025510u))
#define GPIO_PORTF_DEN_R   (*((volatile uint32_t *)0x4002551Cu))
#define GPIO_PORTF_AMSEL_R (*((volatile uint32_t *)0x40025528u))

#define NVIC_EN0_R         (*((volatile uint32_t *)0xE000E100u))
#define NVIC_UNPEND0_R     (*((volatile uint32_t *)0xE000E280u))

#define GPIO_PORTF_RED_DATA_R \
    (*((volatile uint32_t *)0x40025008u))
#define GPIO_PORTF_SW1_DATA_R \
    (*((volatile const uint32_t *)0x40025040u))

#define PORTF_CLOCK    (1u << 5)
#define PORTF_IRQ_MASK (1u << 30)
#define LED_RED        (1u << 1)
#define SW1            (1u << 4)

volatile uint32_t led_state;
volatile uint32_t interrupt_count;

void GPIOPortFIntHandler(void)
{
    if ((GPIO_PORTF_MIS_R & SW1) != 0u)
    {
        GPIO_PORTF_ICR_R = SW1;

        led_state ^= LED_RED;
        GPIO_PORTF_RED_DATA_R = led_state;
        interrupt_count++;
    }
}

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
    GPIO_PORTF_RED_DATA_R = 0u;

    GPIO_PORTF_IS_R &= ~SW1;
    GPIO_PORTF_IBE_R &= ~SW1;
    GPIO_PORTF_IEV_R &= ~SW1;
    GPIO_PORTF_ICR_R = SW1;
    GPIO_PORTF_IM_R |= SW1;

    NVIC_UNPEND0_R = PORTF_IRQ_MASK;
    NVIC_EN0_R = PORTF_IRQ_MASK;

    while (1)
    {
    }
}
```

The startup file supplies the vector table entry for `GPIOPortFIntHandler`. Some CCS startup files initially point every peripheral vector to `IntDefaultHandler`. If yours does, add this declaration in the external interrupt-handler declarations section:

```c
extern void GPIOPortFIntHandler(void);
```

Then change only the vector-table entry commented `GPIO Port F`:

```c
GPIOPortFIntHandler,                      // GPIO Port F
```

Do not rename `IntDefaultHandler` itself or change the other vector entries. The function in `main.c` must be named `GPIOPortFIntHandler` and must not be `static`.

The main loop is empty because the GPIO interrupt does the work. `interrupt_count` is a debugger aid. It is `volatile` so the compiler does not assume it stays unchanged while the CPU is running the ISR.

## CCS steps

1. Open your CCS project and replace `main.c` with the code above.
2. Build the project. If the handler name causes a link error, check the startup file's Port F vector entry.
3. Start **Debug As → Code Composer Debug Session**.
4. Add `led_state` and `interrupt_count` to **Expressions**.
5. Set a breakpoint on the line `led_state ^= LED_RED;` inside `GPIOPortFIntHandler`.
6. Resume the program and press SW1. CCS should stop at the breakpoint.
7. Step over the XOR and inspect `led_state`. Resume, then press SW1 again.

At the breakpoint, the processor has paused in the ISR. After stepping over the XOR, `led_state` should alternate between `0x02` and `0x00`. `interrupt_count` increases for each detected press edge.

## Expected result

| Action | `led_state` after ISR | `interrupt_count` | Red LED |
|---|---:|---:|---|
| Start | `0x00` | `0` | Off |
| Press SW1 once | `0x02` | `1` | On |
| Press SW1 again | `0x00` | `2` | Off |

The counter can increase by more than one for a single physical press. Real buttons can bounce, briefly creating extra edges. We will handle debouncing in a later lesson.

## DriverLib comparison

DriverLib can configure the same interrupt path:

```c
SysCtlPeripheralEnable(SYSCTL_PERIPH_GPIOF);
while (!SysCtlPeripheralReady(SYSCTL_PERIPH_GPIOF))
{
}

GPIOPinTypeGPIOInput(GPIO_PORTF_BASE, GPIO_PIN_4);
GPIOPadConfigSet(GPIO_PORTF_BASE, GPIO_PIN_4,
                 GPIO_STRENGTH_2MA, GPIO_PIN_TYPE_STD_WPU);
GPIOPinTypeGPIOOutput(GPIO_PORTF_BASE, GPIO_PIN_1);

GPIOIntDisable(GPIO_PORTF_BASE, GPIO_PIN_4);
GPIOIntTypeSet(GPIO_PORTF_BASE, GPIO_PIN_4, GPIO_FALLING_EDGE);
GPIOIntClear(GPIO_PORTF_BASE, GPIO_PIN_4);
GPIOIntRegister(GPIO_PORTF_BASE, GPIOPortFIntHandler);
IntEnable(INT_GPIOF);
GPIOIntEnable(GPIO_PORTF_BASE, GPIO_PIN_4);
```

In the handler, use `GPIOIntStatus(GPIO_PORTF_BASE, true)` to read the masked status and `GPIOIntClear(GPIO_PORTF_BASE, GPIO_PIN_4)` to clear the source. The direct-register example makes the GPIO and NVIC steps visible.

## Your notes

- My prediction:
- Did the ISR run when SW1 was pressed?:
- Values seen for `led_state` and `interrupt_count`:
- Did one press ever count more than once?:
- What is still unclear?:

## Short check

1. What edge does the program detect on PF4?
2. Which register lets the Port F interrupt reach the CPU?
3. Why does the ISR write `SW1` to `GPIO_PORTF_ICR_R`?
4. What does `led_state ^= LED_RED` change?
5. Why does the main loop not need to read SW1?

## Questions we discussed

### When does the ISR run?

**Question:** Will the ISR run while SW1 is held down, or when SW1 is first pressed?

**Answer:** It runs when SW1 is first pressed. PF4 changes from high to low, creating the falling edge. Holding the button does not create another edge. Button bounce can create extra edges.

### Why is Port F IRQ 30?

**Question:** How do we know Port F's interrupt number is 30?

**Answer:** The TM4C123GH6PM data sheet assigns GPIO Port F interrupt number 30. `NVIC_EN0` covers interrupt numbers 0–31, so Port F uses bit 30: `1u << 30`, or `0x40000000`.

**Where to find it:** Printed page 105, Table 2-9, “Interrupts (continued).” Find interrupt number 30; its description is GPIO Port F. That row also lists vector number 46.

### How does the CPU reach `GPIOPortFIntHandler`?

**Question:** How does the CPU know to run this function, and is its name standard?

**Answer:** The CPU does not look up a C function by name. When IRQ 30 occurs, the Cortex-M4 uses vector number 46 (IRQ number plus 16) to select an entry in the vector table. That entry contains the handler's address. The startup file and linker connect the vector-table entry to the C function.

`GPIOPortFIntHandler` is not a C or ARM-mandated name. It is the name expected by the startup file in this CCS project. TivaWare CCS startup files commonly use this name, but check your project's vector table and use the exact name it references.

In the startup file we examined, Port F's vector initially points to `IntDefaultHandler`. Add the `extern` declaration for `GPIOPortFIntHandler` and replace only the Port F vector entry with that name. Keep the handler definition in `main.c` named `GPIOPortFIntHandler`.

**If SW1 does not reach the handler:**

1. Make sure the debugger is running, then set a breakpoint inside `GPIOPortFIntHandler` and press SW1.
2. In the startup file, find the Port F vector entry. Confirm that it refers to `GPIOPortFIntHandler`, not a default handler.
3. If the breakpoint is not hit, inspect `GPIO_PORTF_RIS` at `0x40025414`. PF4's bit (`0x10`) should be set after its falling edge.
4. If PF4's bit is set in `GPIO_PORTF_RIS` but not `GPIO_PORTF_MIS` at `0x40025418`, check that `GPIO_PORTF_IM` bit 4 is enabled. If it is set in `GPIO_PORTF_MIS` but the handler is not reached, check `NVIC_EN0` bit 30 and the vector-table entry.

### Why can the interrupt run while the red LED stays off?

**Question:** The handler runs, but the red LED does not turn on. What is wrong?

**Answer:** Check the PF1 direction bit. This line sets PF1 as an output:

```c
GPIO_PORTF_DIR_R |= LED_RED;
```

But this following line is incorrect:

```c
GPIO_PORTF_DIR_R &= SW1;
```

It keeps only bits that are set in `SW1` (`0x10`) and clears PF1's bit (`0x02`). To make PF4 an input without disturbing other pins, clear only PF4:

```c
GPIO_PORTF_DIR_R &= ~SW1;
```

There is a separate address typo to fix: `GPIO_PORTF_MIS_R` is at `0x40025418`, not `0x40025410` (that is `GPIO_PORTF_IM`). With the wrong address, the handler reads the interrupt mask instead of the pending interrupt status.

### Why read the button state in the both-edge exercise?

**Question:** If the original XOR toggle logic still works with both edges enabled, why use this instead?

**Answer:** The XOR version is not inherently wrong. With one clean falling edge on press and one rising edge on release, it toggles on both events and can produce the same visible result. But XOR means “flip the LED on every interrupt.” The exercise goal is for the LED to **follow the button**.

Reading PF4 gives the current button state: low means pressed, so assign `LED_RED`; high means released, so assign `0u`. This sets the LED to the intended state instead of relying on its previous state. It is more robust if the initial LED state is unknown or button bounce causes extra interrupts. The counter increases on each detected edge, so with both-edge mode it can count both press and release events, plus any bounce.

### How do I write a write-one-to-clear register?

**Question:** For a W1C register, should I use `|=` or write the mask directly?

**Answer:** Write the mask directly. In a W1C register, writing `1` clears that flag and writing `0` leaves it unchanged:

```c
GPIO_PORTF_ICR_R = SW1;          // Clear only the SW1 flag
GPIO_PORTF_ICR_R = SW1 | SW2;    // Clear both flags
```

Avoid `|=` and `&=` here. They perform read-modify-write operations, which are not the right way to acknowledge W1C flags. Check the datasheet register type and write description before choosing an operation; ordinary read/write registers and W1C registers follow different rules.

## Exercises

### Exercise 1 — Count without toggling

**Goal:** Count each detected SW1 press, but leave the red LED off.

**Solution:** In `GPIOPortFIntHandler`, remove the two LED lines and keep the counter:

```c
GPIO_PORTF_ICR_R = SW1;
interrupt_count++;
```

Keep the initial write `GPIO_PORTF_RED_DATA_R = 0u;` in `main`.

**Check:** Set a breakpoint after `interrupt_count++`. Each detected press should increase the count. A single physical press may create extra counts because there is no debounce yet.

### Exercise 2 — Toggle the blue LED

**Goal:** Keep the interrupt behavior, but toggle the blue LED instead.

**Solution:** Use PF2's data alias and mask:

```c
#define GPIO_PORTF_BLUE_DATA_R \
    (*((volatile uint32_t *)0x40025010u))
#define LED_BLUE (1u << 2)
```

In `main`, replace `LED_RED` with `LED_BLUE` in the AFSEL, AMSEL, and DIR setup. Also replace the LED bit in DEN and initialize the blue output to zero. In the ISR, use:

```c
led_state ^= LED_BLUE;
GPIO_PORTF_BLUE_DATA_R = led_state;
```

**Check:** Each SW1 press should alternate the blue LED. The red LED should stay off.

### Exercise 3 — Turn the LED on at press and off at release

**Goal:** Use an interrupt on both PF4 edges. The LED should follow the button state.

**Solution:** In `main`, enable both-edge detection:

```c
GPIO_PORTF_IBE_R |= SW1;
```

Replace the ISR's toggle code with:

```c
GPIO_PORTF_ICR_R = SW1;
led_state =
    (GPIO_PORTF_SW1_DATA_R == 0u) ? LED_RED : 0u;
GPIO_PORTF_RED_DATA_R = led_state;
interrupt_count++;
```

Leave `GPIO_PORTF_IS_R` cleared for edge detection. With both-edge detection enabled, `GPIO_PORTF_IEV_R` does not select the edge.

**Check:** The LED should turn on when SW1 is pressed and off when it is released. The ISR should run on both transitions.

### Exercise 4 — Toggle green with SW2

**Goal:** Use SW2 (PF0) to toggle the green LED (PF3).

**Solution:** Add these definitions:

```c
#define GPIO_PORTF_LOCK_R (*((volatile uint32_t *)0x40025520u))
#define GPIO_PORTF_CR_R   (*((volatile uint32_t *)0x40025524u))
#define GPIO_PORTF_GREEN_DATA_R \
    (*((volatile uint32_t *)0x40025020u))
#define GPIO_PORTF_SW2_DATA_R \
    (*((volatile const uint32_t *)0x40025004u))

#define GPIO_LOCK_KEY 0x4C4F434Bu
#define SW2 (1u << 0)
#define LED_GREEN (1u << 3)
```

After Port F is ready, unlock and commit PF0 before configuring it:

```c
GPIO_PORTF_LOCK_R = GPIO_LOCK_KEY;
GPIO_PORTF_CR_R |= SW2;
```

Configure PF0 as an input with pull-up and digital enable. Configure PF3 as a digital output. Set GPIO edge detection for `SW2` as falling edge, clear its flag, and unmask it:

```c
GPIO_PORTF_AFSEL_R &= ~(SW2 | LED_GREEN);
GPIO_PORTF_AMSEL_R &= ~(SW2 | LED_GREEN);
GPIO_PORTF_DIR_R &= ~SW2;
GPIO_PORTF_DIR_R |= LED_GREEN;
GPIO_PORTF_PUR_R |= SW2;
GPIO_PORTF_DEN_R |= SW2 | LED_GREEN;

GPIO_PORTF_IS_R &= ~SW2;
GPIO_PORTF_IBE_R &= ~SW2;
GPIO_PORTF_IEV_R &= ~SW2;
GPIO_PORTF_ICR_R = SW2;
GPIO_PORTF_IM_R |= SW2;
```

In the ISR, check and clear SW2, then toggle green:

```c
if ((GPIO_PORTF_MIS_R & SW2) != 0u)
{
    GPIO_PORTF_ICR_R = SW2;
    led_state ^= LED_GREEN;
    GPIO_PORTF_GREEN_DATA_R = led_state;
}
```

The Port F NVIC interrupt is already enabled by the main lesson.

**Check:** Pressing SW2 should toggle green. PF0 must be unlocked after Port F is ready and before changing its GPIO settings.

### Exercise 5 — Handle SW1 and SW2 independently

**Goal:** Make SW1 toggle red and SW2 toggle blue, using one Port F ISR.

**Solution:** Enable PF0 as in Exercise 4. Set up PF4 and PF0 as falling-edge interrupt inputs, enable both in `GPIO_PORTF_IM_R`, and configure PF1 and PF2 as outputs. Unlock and commit PF0 before configuring it. Use both interrupt masks:

```c
#define LED_BLUE (1u << 2)

GPIO_PORTF_ICR_R = SW1 | SW2;
GPIO_PORTF_IM_R |= SW1 | SW2;
```

In the ISR, read the pending pin status once. Clear only the pending sources, then update each LED independently:

```c
void GPIOPortFIntHandler(void)
{
    uint32_t status = GPIO_PORTF_MIS_R;
    GPIO_PORTF_ICR_R = status;

    if ((status & SW1) != 0u)
    {
        led_state ^= LED_RED;
        GPIO_PORTF_RED_DATA_R =
            (led_state & LED_RED) ? LED_RED : 0u;
    }

    if ((status & SW2) != 0u)
    {
        led_state ^= LED_BLUE;
        GPIO_PORTF_BLUE_DATA_R =
            (led_state & LED_BLUE) ? LED_BLUE : 0u;
    }
}
```

Configure PF1 and PF2 as digital outputs, and initialize `led_state` and both LED outputs to zero. Keep the Port F NVIC enable from the main lesson.

**Check:** SW1 should toggle only red; SW2 should toggle only blue. Watch `led_state`: red uses bit 1 (`0x02`) and blue uses bit 2 (`0x04`).

## Next lesson

Next, we will study the Cortex-M4 exception and NVIC behavior more closely, including interrupt priorities and what the CPU saves when an interrupt occurs.
