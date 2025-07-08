05_Amazon -: LDD
============================
01. Explain prob function and how driver get intialize.
02. Explain I2C and SPI full details (HW,Prococol specific, Modes, Complexity)
03. Kernel Panic and how you debug.
04. Any driver from End to End and how it works and what are apis we can use.
05. Board Bringup for custom board (IF know secure boot that good)
06. RootFS bringup for Any board.
07. ARM Dispatcher and schedular
08. Configure 1 pin for input and one for output. Input pin as interrrupt.
    How you will define top half and bottom half.
09. What is ISR. How it works. What we shoud not do in ISR.
10. Way to access from userspace to kernel space.
===================================================================================

# 05\_Amazon — LDD: Deep Answers from Scratch

> This guide is structured exactly to your 10 prompts, with mini block‑diagrams, tight code (commented for action + outcome), and checklists you can quote in interviews.

---

## 01) **Probe function & how a Linux driver gets initialized**

### Big picture (bus → match → probe)

```
        Device Tree / ACPI / hotplug
                   │
                   ▼
   bus_type (platform/i2c/spi/usb/pci)
        ├─ matches `compatible` / IDs
        ▼
   driver->probe(dev)
        ├─ map regs / request IRQ / clocks / regulators
        ├─ create kobjects (/sys), char devs (/dev)
        └─ ready for userspace
```

### Why “probe”?

* Kernel may have many drivers for many devices on the same bus. When the bus core **matches** a device with a driver (via `of_match_table`, `id_table`, ACPI IDs, or modalias), it calls the driver’s **`probe()`** to **bind** resources and become operational. `remove()` undoes it.

### Minimal platform driver skeleton

```c
// drivers/misc/my_dev.c
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>

static int my_probe(struct platform_device *pdev)
{
    // ACTION: Get resources, allocate state, register char/sysfs
    // OUTCOME: Device is usable after probe returns 0
    return 0;
}

static int my_remove(struct platform_device *pdev)
{
    // ACTION: Free resources allocated in probe
    return 0;
}

static const struct of_device_id my_of_match[] = {
    { .compatible = "myco,my-dev" },
    { }
};
MODULE_DEVICE_TABLE(of, my_of_match);

static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = {
        .name           = "my-dev",
        .of_match_table = my_of_match,
    },
};

module_platform_driver(my_driver);
MODULE_LICENSE("GPL");
```

**Init path recap**

1. Boot loads DT/ACPI → bus core creates `struct device`.
2. udev may auto-load module by **modalias**.
3. Bus matches → `probe()`.
4. Driver uses `devm_*` helpers to auto-clean on `remove()`/unwind.

---

## 02) **I²C vs SPI — HW, protocol, modes, complexity**

### Quick table

| Aspect      | I²C                                       | SPI                                                 |
| ----------- | ----------------------------------------- | --------------------------------------------------- |
| Lines       | 2: SDA, SCL (open‑drain + pull‑ups)       | 3+ : SCLK, MOSI, MISO, + per‑slave CS               |
| Topology    | Multi‑master, shared bus                  | Single master (typical), point‑to‑point per CS      |
| Addressing  | 7/10‑bit addresses                        | No addressing; chip select picks device             |
| Speed       | 100k/400k/1MHz/3.4MHz (Std/Fast/Fast+/HS) | Up to 10s of MHz (common 10–50 MHz; higher on SoCs) |
| Transfer    | Half‑duplex, byte‑based, ACK/NACK         | Full‑duplex stream; mode via CPOL/CPHA              |
| Arbitration | Yes (multi‑master)                        | No                                                  |
| Complexity  | Simpler wiring, protocol overhead         | More wires, simpler protocol                        |

### Linux I²C stack (adapter ↔ client)

```
I2C controller driver (adapter)
        ▲            ▲
        │            └─ provides xfer/smbus ops
        │
I2C client driver (your device)
```

**Device Tree**

```dts
&i2c1 {
    status = "okay";
    temp@48 {
        compatible = "myco,my-temp";
        reg = <0x48>;                 // I2C address
        interrupt-parent = <&gpio0>;
        interrupts = <12 IRQ_TYPE_EDGE_RISING>;
    };
};
```

**Key APIs**

* SMBus helpers: `i2c_smbus_read_byte_data()`, `_write_byte_data()`, etc.
* Raw: `i2c_transfer()`, `i2c_master_send()`, `i2c_master_recv()`.
* Get `struct i2c_client *client` in `probe()`; stash drvdata via `i2c_set_clientdata()`.

### Linux SPI stack (controller ↔ device)

```
SPI controller (master) driver
        ▲
        │ registers `spi_controller`
        ▼
SPI device (your peripheral) driver
```

**Device Tree**

```dts
&spi0 {
    status = "okay";
    myflash@0 {
        compatible = "myco,myspi";
        reg = <0>;                     // CS index
        spi-max-frequency = <25000000>;
        spi-cpol;                      // CPOL=1 (optional)
        spi-cpha;                      // CPHA=1 (optional)
        // mode = 0..3 is CPOL/CPHA combo
    };
};
```

**Key APIs**

* Simple: `spi_write()`, `spi_read()`, `spi_write_then_read()`.
* Advanced: `spi_message`, `spi_transfer`, `spi_sync()`, `spi_async()`.
* Device params: `spi->mode` (CPOL/CPHA), `bits_per_word`, `max_speed_hz`.

**Choosing**

* Few wires + many slow sensors → I²C.
* High‑throughput/streaming (displays, ADCs, flash) → SPI.

---

## 03) **Kernel panic & how to debug**

### Know the types

* **Oops**: recoverable fault; may kill offending task.
* **Panic**: unrecoverable; system halts/reboots (`panic_on_oops` can escalate oops → panic).

### First steps

1. **Capture logs**: serial console, `pstore`, `netconsole`, or kdump.
2. **Read the trace**: look at **panic string**, **call trace**, **PC/LR**, **faulting address**, **BUG/WARN**.
3. **Map addresses**: `addr2line -e vmlinux 0xffffffc0100abcd`.

### Toolbox

* Build options: `CONFIG_DEBUG_INFO`, `CONFIG_KASAN`, `CONFIG_KCSAN`, `CONFIG_KMEMLEAK`, `CONFIG_LOCKDEP`.
* Tracing: `ftrace` (`echo function_graph > current_tracer`), `trace-cmd`, `perf`.
* Live debug: **KGDB/KDB**, JTAG.
* Post‑mortem: **kdump** + `crash` utility.
* Logging: `dynamic_debug`, `pr_debug`, `dev_dbg`, rate‑limit prints.

### Methodical flow

```
Reproduce → Capture (kdump/serial) → Localize (stack) →
Minimize (bisect / disable suspects) → Inspect (addr2line) →
Instrument (ftrace/KASAN/lockdep) → Fix → Prove via stress
```

Common root causes: NULL deref, use‑after‑free, bad IRQ handling, stack overflow, wrong clocks/regulators, DMA mapping mistakes, races.

---

## 04) **Any driver end‑to‑end: example with I²C sensor + char dev**

**Goal**: expose a temperature sensor over `/dev/mytemp0` with blocking read and an IRQ for "data ready".

### Device Tree

```dts
&i2c1 {
    mytemp@48 {
        compatible = "myco,mytemp";
        reg = <0x48>;
        drdy-gpios = <&gpio0 12 GPIO_ACTIVE_HIGH>;
        interrupt-parent = <&gpio0>;
        interrupts = <12 IRQ_TYPE_EDGE_RISING>;
    };
};
```

### Driver core (highlights)

```c
struct mytemp {
    struct i2c_client *client;
    int irq;
    wait_queue_head_t wq;  // for blocking read
    bool data_ready;
    dev_t devt;
    struct cdev cdev;
    struct class *class;
};

static irqreturn_t mytemp_irq(int irq, void *data)
{
    struct mytemp *st = data;
    st->data_ready = true;          // ACTION: flag event
    wake_up_interruptible(&st->wq); // OUTCOME: wake blocking readers
    return IRQ_HANDLED;             // (top half is minimal)
}

static ssize_t my_read(struct file *f, char __user *ubuf, size_t len, loff_t *off)
{
    struct mytemp *st = f->private_data;
    int ret; u8 reg = 0x00; u8 buf[2];

    // Block until IRQ says data_ready
    wait_event_interruptible(st->wq, st->data_ready);
    st->data_ready = false;

    ret = i2c_smbus_read_i2c_block_data(st->client, reg, 2, buf);
    if (ret < 0) return ret;

    if (len < 2) return -EINVAL;
    if (copy_to_user(ubuf, buf, 2)) return -EFAULT;
    return 2;
}

static int my_open(struct inode *ino, struct file *f)
{
    f->private_data = container_of(ino->i_cdev, struct mytemp, cdev);
    return 0;
}

static const struct file_operations my_fops = {
    .owner = THIS_MODULE,
    .read  = my_read,
    .open  = my_open,
    .llseek = no_llseek,
};

static int my_probe(struct i2c_client *client, const struct i2c_device_id *id)
{
    struct mytemp *st; int ret; int irq;

    st = devm_kzalloc(&client->dev, sizeof(*st), GFP_KERNEL);
    if (!st) return -ENOMEM;
    st->client = client;
    init_waitqueue_head(&st->wq);

    irq = gpiod_to_irq(devm_gpiod_get(&client->dev, "drdy", GPIOD_IN));
    if (irq < 0) return irq;
    st->irq = irq;

    ret = devm_request_threaded_irq(&client->dev, st->irq,
            NULL, mytemp_irq, IRQF_ONESHOT | IRQF_TRIGGER_RISING,
            dev_name(&client->dev), st);
    if (ret) return ret;

    // Char device (/dev/mytemp0)
    ret = alloc_chrdev_region(&st->devt, 0, 1, "mytemp");
    if (ret) return ret;
    cdev_init(&st->cdev, &my_fops);
    st->cdev.owner = THIS_MODULE;
    ret = cdev_add(&st->cdev, st->devt, 1);
    if (ret) goto unreg;
    st->class = class_create(THIS_MODULE, "mytemp");
    device_create(st->class, NULL, st->devt, NULL, "mytemp0");

    i2c_set_clientdata(client, st);
    return 0;

unreg:
    unregister_chrdev_region(st->devt, 1);
    return ret;
}

static int my_remove(struct i2c_client *client)
{
    struct mytemp *st = i2c_get_clientdata(client);
    device_destroy(st->class, st->devt);
    class_destroy(st->class);
    unregister_chrdev_region(st->devt, 1);
    return 0;
}

static const struct of_device_id my_of[] = {
    { .compatible = "myco,mytemp" }, { }
};
MODULE_DEVICE_TABLE(of, my_of);

static struct i2c_driver my_driver = {
    .driver = { .name = "mytemp", .of_match_table = my_of },
    .probe_new = my_probe,
    .remove = my_remove,
};
module_i2c_driver(my_driver);
MODULE_LICENSE("GPL");
```

**User app**

```c
// blocks until data ready, then reads 2 bytes
int fd = open("/dev/mytemp0", O_RDONLY);
uint8_t buf[2];
read(fd, buf, 2);
```

APIs used: `devm_*`, `gpiod_*`, `devm_request_threaded_irq`, `wait_event_interruptible`, `i2c_smbus_*`, cdev APIs.

---

## 05) **Board bring‑up for a custom board (incl. Secure Boot basics)**

### Checklist

1. **Power & clocks**: Verify rails, resets, oscillator, PLLs; check PMIC sequences with scope.
2. **Boot pins/straps**: Ensure boot device (QSPI/eMMC/SD) selected.
3. **UART console**: First life — enable in BootROM/U‑Boot DT, match pins in pinctrl.
4. **DRAM init**: In SPL/TF‑A; tune timings; run memtests.
5. **Bootloader port**: Configure drivers for storage, I²C (PMIC), GPIO, Ethernet.
6. **Device Tree**: Create base SoC dtsi + board dts; describe clocks, regulators, pinmux, buses, PHYs, sensors.
7. **Kernel config**: Start from SoC defconfig; enable required drivers, filesystems, networking.
8. **Peripherals**: Bring up one at a time; use `devicetree` overlays only if needed.
9. **Diagnostics**: LEDs, GPIO loopbacks, I²C scan, SPI loopback, Ethernet ping.

### Secure Boot (high level)

* **Root of Trust in ROM** verifies next stage (TF‑A/SPL) using a **public key** burned/fused.
* Boot chain: **ROM → (TF‑A/SPL) → U‑Boot → Kernel → RootFS**; each stage **verifies signature** of the next.
* Practical:

  * Generate offline **key pair**; burn **hash of public key** into fuses.
  * Sign images (SPL/U‑Boot/fitImage) → enforce verification in bootloader.
  * Lock debug ports or require auth for JTAG.

---

## 06) **RootFS bring‑up for any board**

### Minimal BusyBox rootfs (quick path)

1. Build BusyBox (static or dynamic). Install to `rootfs/`.
2. Create dirs: `dev proc sys tmp run etc mnt root bin sbin usr/lib var`.
3. **Init**: `/sbin/init` (BusyBox) or `systemd`.
4. Devices: use `mdev` or `udev`.
5. `/etc/inittab` or systemd units to start getty on console.
6. Add `/etc/fstab` for mounting `proc`, `sysfs`, `tmpfs`.
7. Pack as ext4 (or squashfs+overlay): `mkfs.ext4 rootfs.img`, copy to eMMC/SD.

### Yocto/Buildroot (scalable path)

* Select **SoC BSP layer**; set **MACHINE**; enable recipes for kernel modules, firmware, debug tools; add your packages; ensure dtb + kernel deploy into image.

### Early debug

* Use **init=/bin/sh** on kernel cmdline.
* Ensure console: `console=ttyS0,115200` (match your UART).

---

## 07) **ARM “dispatcher” & Linux scheduler**

### ARM exception/interrupt dispatch (AArch64)

```
EL3 (Secure Monitor)  ┐
EL2 (Hypervisor)      ├─ Vector table: sync/irq/fiq/serror at each EL
EL1 (Kernel)          ┘
EL0 (User)
```

* Interrupt arrives → GIC routes to CPU → **vector** entry saves context → kernel’s **generic IRQ layer** runs:

  1. mask/ack → 2) `handle_irq_event()` → 3) your ISR (top‑half) → 4) threaded/bottom‑half if used.

### Linux scheduler (CFS + RT + DL)

* Classes: `stop` > `rt` > `dl` > `fair(CFS)` > `idle` (priority order).
* **CFS**: per‑CPU runqueue, virtual runtime, preemption; NUMA/affinity aware.
* **RT**: fixed priorities; `SCHED_FIFO`/`RR`.
* **Deadline**: `SCHED_DEADLINE` for EDF.
* Preemption: voluntary/preemptible kernels; **tickless** reduces overhead.

---

## 08) **Configure one pin input (IRQ) + one pin output; top/bottom halves**

### Device Tree

```dts
pinctrl: pinctrl@... { /* SoC-specific pinmux omitted */ };

gpio-led {
    compatible = "gpio-leds";
    led0: led0 { gpios = <&gpio2 5 GPIO_ACTIVE_HIGH>; };
};

button@0 {
    compatible = "gpio-keys";
    button0: button0 {
        label = "btn0";
        gpios = <&gpio0 12 GPIO_ACTIVE_LOW>;
        linux,code = <KEY_ENTER>;
        interrupt-parent = <&gpio0>;
        interrupts = <12 IRQ_TYPE_EDGE_BOTH>;
    };
};
```

### In your driver (gpiod + IRQ)

```c
struct mygpio {
    struct gpio_desc *led;
    int irq;
    struct work_struct work; // bottom half via workqueue
};

static irqreturn_t my_top(int irq, void *arg)
{
    struct mygpio *st = arg;
    // ACTION: keep it tiny; schedule bottom half
    schedule_work(&st->work);
    return IRQ_HANDLED;
}

static void my_bottom(struct work_struct *w)
{
    struct mygpio *st = container_of(w, struct mygpio, work);
    // OUTCOME: do slow/sleepable work here
    gpiod_set_value_cansleep(st->led, 1);
    msleep(10);
    gpiod_set_value_cansleep(st->led, 0);
}

static int probe(...){
    struct mygpio *st = devm_kzalloc(...);
    st->led = devm_gpiod_get(&pdev->dev, "led", GPIOD_OUT_LOW);
    INIT_WORK(&st->work, my_bottom);
    st->irq = platform_get_irq(pdev, 0);

    return devm_request_irq(&pdev->dev, st->irq, my_top,
            IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING,
            dev_name(&pdev->dev), st);
}
```

**Top half** = ISR: quick, no sleeping. **Bottom half** = tasklet/workqueue/threaded IRQ: can sleep/do longer work.

---

## 09) **What is ISR? What not to do in ISR**

**ISR (Interrupt Service Routine)** handles an IRQ quickly so the system can resume.

**Do**

* Acknowledge/clear HW interrupt.
* Capture minimal state; schedule bottom half.
* Use `spin_lock_irqsave()` only briefly.

**Don’t**

* **Sleep/block** (no `mutex`, `msleep`, `copy_to_user`, I²C/SPI unless using **threaded IRQ**).
* Do heavy logging or long loops.
* Hold spinlocks long; call memory alloc with `GFP_KERNEL` (use `GFP_ATOMIC` if unavoidable).
* Touch userspace or slow buses; avoid re‑enabling same IRQ too early.

Prefer **`devm_request_threaded_irq()`** to keep top‑half tiny and do work in the thread handler.

---

## 10) **Ways for userspace ↔ kernelspace**

### Options & when to use

* **Char device (`/dev/foo`)**: custom protocol; `read/write/ioctl/poll/mmap`.
* **Sysfs (`/sys/...`)**: small attributes (text), one value per file.
* **Debugfs**: developer diagnostics (non‑ABI).
* **Netlink**: structured messaging, multicast (e.g., networking cfg, generic‑netlink).
* **UIO/VFIO**: map device regions to userspace safely.
* **`ioctl`**: control ops; prefer netlink/sysfs if simple.
* **`mmap`**: share DMA buffers.

### Mini char‑dev with poll (event + data)

```c
static DECLARE_WAIT_QUEUE_HEAD(wq);
static int event;

static unsigned int my_poll(struct file *f, poll_table *wait)
{
    poll_wait(f, &wq, wait);
    return event ? (POLLIN | POLLRDNORM) : 0;
}

static ssize_t my_read(struct file *f, char __user *ubuf, size_t len, loff_t *off)
{
    if (!event)
        if (wait_event_interruptible(wq, event))
            return -ERESTARTSYS;
    event = 0;
    // copy data to userspace...
    return 0;
}

// From IRQ bottom half:
static void on_data_ready(void)
{
    event = 1;
    wake_up_interruptible(&wq);
}
```

**Userspace** can then `select()/poll()` the fd and `read()` when ready.

---

## Rapid interview one‑liners (for recall)

* **Probe**: called on match; allocate, request IRQ/regs, expose interfaces; `remove()` frees.
* **I²C vs SPI**: I²C = 2‑wire addressed half‑duplex; SPI = 4‑wire CS‑based full‑duplex, higher throughput.
* **Panic debug**: capture → stack → addr2line → ftrace/KASAN/lockdep → fix → stress.
* **Top vs bottom**: ISR tiny + non‑sleep; bottom half does the work.
* **Userspace links**: char dev for rich I/O; sysfs for attributes; netlink for structured control.

---

### Final tips

* Prefer **`devm_*`** for resource life‑cycle.
* Rate‑limit logs; expose knobs via **sysfs** or **debugfs**.
* Always document DT bindings & ABI.
* Add selftests in `tools/testing/` or `kunit` where feasible.
