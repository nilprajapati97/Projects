Eximietas
=================

01. Custom Board bringup.When there is not any info about board.
02. SAD interrupt.
03. What is GIC and How it works flow.
04. You have one pin for interrupt for processor. And how it will come to main SOC. Explain in details.
05. You have interrupt ISR and Pin you configured interrupt. But somewhere its not calling ISR.
    Explain debug flow.
06. You have SPI driver nad device. So MOSI is getting straight line. What is your approch and fault.
07. inany project what most challenging roll you have faced and How you handled it.
08. Write calculate sige of any variable witout use sizeoff operator.
==============================================================================================


Great — these are **classic deep-dive questions** you’ll get in a **Board Support Package (BSP) + Device Driver + System Bring-up** interview at 10+ years experience.
Let’s break each one down, **in-depth** with **debug flow + examples** so you can use them in interviews 👇

---

## **01. Custom Board Bring-up (No Info about Board)**

👉 Scenario: You receive a **custom hardware board** (new SoC + peripherals), but no documentation.

**Approach:**

1. **Start at Hardware Basics**

   * Check processor/SoC part number → search public datasheet.
   * Trace board schematics (if available) → power rails, clock source, reset pin, boot pins.

2. **Boot Flow**

   * Identify BootROM sequence of SoC (usually internal ROM loads SPL/bootloader).
   * Attach UART/JTAG to capture early boot logs (even garbage → check baud rate).

3. **Step-by-Step Bring-up**

   * **Power-on check**: measure voltage rails using DSO.
   * **Clocks/Reset**: check if oscillator/PLL signals are stable.
   * **Boot medium**: figure out where BootROM fetches first code (eMMC, SPI-NOR, SD).
   * **UART early print**: enable debug UART in SPL, verify console.
   * **DRAM init**: critical step → configure DDR controller (timings from datasheet).
   * **Peripheral bring-up**: GPIO → I2C/SPI → Ethernet → Display.

**Answer Tip (Interview):**

> “I’d begin by validating power and clock domains, then bring up UART for early debug, then focus on DDR init and boot medium. With no docs, I’d rely on JTAG + oscilloscope + trial SPL code to progressively enable subsystems.”

---

## **02. SAD Interrupt in Linux Kernel**

👉 **SAD = Security Association Database** interrupt (IPSec/crypto engines).

* In SoCs with IPSec hardware accelerator, SAD table triggers interrupt when a packet matches/misses or when rekeying events occur.

**Debug Flow:**

* Check `/proc/interrupts` → confirm IRQ line increments.
* Verify device-tree node (e.g. `interrupts = <0 45 4>;`) is correct.
* Confirm driver registered ISR with `request_irq()`.
* Enable `dynamic_debug` in driver (`echo 'file driver.c +p' > /sys/kernel/debug/dynamic_debug/control`).

---

## **03. What is GIC and How It Works (Flow)**

👉 **GIC = ARM Generic Interrupt Controller** (e.g., GICv2, GICv3).
It routes hardware interrupts → CPU cores.

**Flow:**

1. **Peripheral raises interrupt** on its IRQ line.
2. **Interrupt line → SoC interrupt aggregator → GIC Distributor.**
3. **GIC Distributor**:

   * Maps IRQ ID to CPU core(s).
   * Applies priority, masking, targeting rules.
4. **GIC CPU Interface**:

   * Sends IRQ signal into CPU core.
   * CPU reads **ICC\_IARx\_EL1** register to know IRQ ID.
5. **Kernel ISR flow**:

   * `do_IRQ()` → `irq_enter()` → handler → `irq_exit()`.

---

## **04. One Pin Interrupt → How it Reaches SoC**

**Example:** GPIO pin → Interrupt path.

1. Device raises logic level on GPIO pin.
2. GPIO controller detects edge (configured in register).
3. GPIO controller asserts IRQ line → SoC interrupt crossbar.
4. SoC maps GPIO IRQ to GIC input.
5. GIC forwards to CPU core → ISR invoked.

**Interview Explanation:**

> “Even if only one physical pin is visible, it is routed internally via GPIO block → interrupt controller → GIC → CPU. The software must configure pin mux, GPIO edge polarity, and GIC routing.”

---

## **05. ISR Not Called Debug Flow**

👉 You configured interrupt pin but ISR not firing.

**Checklist:**

1. **Pinmux** – confirm pin configured as GPIO-IRQ (not UART/SPI).
2. **Interrupt Type** – edge vs level mismatch? (SoC may require `rising_edge` but board wired as `falling`).
3. **GPIO/IRQ enable** – check `cat /proc/interrupts` if counter increases.
4. **GIC Routing** – confirm interrupt ID enabled in distributor.
5. **Linux Driver** – confirm `request_irq()` returns success.
6. **ISR debugging** – add `pr_info()` logs inside ISR.
7. **Hardware check** – scope the pin; maybe external circuit doesn’t toggle.

---

## **06. SPI Driver – MOSI Straight Line**

👉 Issue: MOSI stays constant (no toggling).

**Causes:**

1. **Clock not running** → check SCK with scope.
2. **Chip Select not asserted** → MOSI won’t toggle without CS low.
3. **Pinmux misconfigured** → MOSI pin may still be GPIO.
4. **Driver init** – check SPI controller enable bit.
5. **Power domain** – peripheral block may be gated.
6. **Loopback test** – send dummy bytes, check SCK/MOSI toggling.

**Debug Flow:**

* `spidev_test` user utility.
* Scope: check SCK, MOSI, CS.
* Kernel logs: ensure `spi_transfer` is called.

---

## **07. Most Challenging Role You Faced**

👉 Interview wants **story + problem + resolution**.

Example Answer (customizable):

> “In one project, during early BSP bring-up, DDR timings were undocumented. The board kept crashing randomly. I set up JTAG, read DDR controller registers, and iteratively tuned timing parameters while comparing with reference boards. Finally stabilized memory training and brought the board to Linux boot. It was the most challenging but also rewarding experience.”

(✔ Shows problem-solving, persistence, low-level expertise).

---

## **08. Calculate Size of Variable Without `sizeof`**

👉 Trick: use pointer arithmetic.

```c
#define SIZEOF(var)    ((char*)(&var+1) - (char*)(&var))

int main() {
    int x;
    printf("Size of int: %d\n", SIZEOF(x));
    double d;
    printf("Size of double: %d\n", SIZEOF(d));
}
```

**Explanation:**

* `&var + 1` → address after variable.
* Cast to `char*` → difference in bytes.

---

✅ So these 8 cover **bring-up, interrupts, debugging, SPI, and tricky C coding.**
Would you like me to also make a **whiteboard block diagram** (Power → GPIO → GIC → CPU → ISR flow) for Q3+Q4? That will help in interviews when explaining.
