Interview Questions
=======================================
01. ISR Api
02. Driver Registration API.
03. ISR flag.
04. What is thread? How it works.
05. Synchronization technique.
06. Difference Spinlock, mutex,semaphore
07. Where we use Spinlock, mutex,semaphore?
08. While we use ISR, which technology we use and why?
09. Difference between malloc and calloc.
10. Stack related interview question and answer.
11. Stack memory of thread and explain.
12. Difference between kmalloc and zmalloc.
13. Which command use for loading and undloading driver module.
14. Differnce between staticall and dynamically load driver.
15. How to add nodes in linux dth. Write and node & configuration.
16. Difference between dts and dtsi file.
17. What is structure.
18. write a program to reverse a string.
19. Write a program to calculate size of structure without using bitwise, Explain 2 3 methods.
20. What is structure padding how to resolve it.
21. What is difference between indmode and probe.
22. What is IOCTL function.
23. Where we use IOCTL function. Explain real scenario.
24. Which modification we need while we add new node in kernel source code.
25. What us sysfs?
26. What is major and minor number is device driver?
27. What is sysfs and proc in linux?
28. Which API used for finding major and minor number?
29. Explain file operation in device driver? How it works in userspace and kernel space?
30. What is stack section?
31. How variable allocating memory? How to access heap?
32. Difference between const & volatile? If we not use volatile than what will happen?
    Explain practical case.
33. What is segmentation fault?
34. What is kernel panic?
35. What is stack overflow?
36. Which APIs used for copy data from user to kernel and kernel to user?
37. Another any api for copy_to_user and copy_from_user?

==========================================================================================================

Awesome list. Here’s a tight, interview-ready set of answers you can skim or deep-dive. I’ve grouped code where it helps and kept explanations “from scratch,” with practical kernel/device-driver context.

---

## 01) ISR API (Linux kernel)

* **Core registration APIs**

  * `int request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags, const char *name, void *dev);`
  * `void free_irq(unsigned int irq, void *dev);`
  * Managed forms (auto-freed at driver detach): `int devm_request_irq(struct device *dev, unsigned int irq, irq_handler_t handler, unsigned long flags, const char *name, void *dev_id);`
* **Handler signatures**

  * `irqreturn_t handler(int irq, void *dev_id);` → return `IRQ_HANDLED` or `IRQ_NONE`.
* **Control helpers**

  * `disable_irq()/enable_irq()`, `disable_irq_nosync()`
  * `irq_set_irq_type(irq, IRQ_TYPE_LEVEL_HIGH/LOW | IRQ_TYPE_EDGE_RISING/FALLING/BOTH)`
* **Threaded IRQ**

  * `request_threaded_irq(irq, top_half, thread_fn, flags, name, dev_id);` → top half runs fast; `thread_fn` runs in kthread context and can sleep if `IRQF_ONESHOT`.

---

## 02) Driver registration API (per bus + char dev)

* **Platform**: `platform_driver_register()` / `module_platform_driver()`, with `.probe/.remove` and `.of_match_table`.
* **PCI**: `pci_register_driver()`.
* **I²C/SPI**: `i2c_add_driver()`, `spi_register_driver()`.
* **Character device**:

  * Allocate device numbers: `alloc_chrdev_region(&devt, first, count, "name")`
  * Init & add: `cdev_init(&cdev, &fops); cdev_add(&cdev, devt, count)`
  * Create node in `/dev`: `class_create()`, `device_create()`.
* **Device Tree binding**: `of_device_id` table + `MODULE_DEVICE_TABLE(of, …)`.

---

## 03) ISR “flags” (the `flags` in `request_irq`)

Common ones:

* `IRQF_SHARED` (shared line; `dev_id` unique per sharer)
* `IRQF_TRIGGER_RISING/FALLING/HIGH/LOW` (edge/level type)
* `IRQF_ONESHOT` (for threaded IRQ; masks line until thread completes)
* `IRQF_NO_THREAD` (force non-threadable)
* `IRQF_PERCPU` (per-CPU IRQ)

---

## 04) What is a thread? How it works

* **Thread** = schedulable execution context sharing a process’s address space.
* **User space**: created via `pthread_create()`; has its own **stack**, registers, TCB; shares heap/data with siblings.
* **Kernel**: each task has `task_struct`; scheduler picks runnable threads; context switch saves/restores CPU state.
* **Kernel threads**: `kthread_run()` create threads that run in kernel space (no user mm).

---

## 05) Synchronization techniques (kernel-centric)

* **Spinlocks** (`spinlock_t`) for short, atomic critical sections; usable in **IRQ context**; no sleeping.
* **Mutexes** (`struct mutex`) for longer sections; **may sleep**; not in IRQ.
* **Semaphores** (`struct semaphore`) counting; rarely used vs mutex; can sleep.
* **RW locks** (`rwlock_t`, `rw_semaphore`) many readers / single writer.
* **Completions**, **wait queues**, **workqueues**, **RCU**, **seqlocks**, **atomic\_t/refcount\_t**, **barriers** (`smp_mb()`), **spin\_lock\_irqsave()\`** (lock + IRQ mask).

---

## 06) Spinlock vs Mutex vs Semaphore

* **Spinlock**: busy-wait; tiny sections; usable in ISR/top half; disables preemption/IRQs as needed.
* **Mutex**: sleeps if contended; only process context; ownership semantics.
* **Semaphore**: counting resource gate (N units); can sleep; weaker ownership semantics.
  **Rule of thumb**: ISR/top-half → spinlock. Sleepable path → mutex. Limited resource pool (N buffers) → semaphore.

---

## 07) Where to use them

* **Spinlock**: protect ring buffer indices in ISR+thread, list heads touched in hard IRQ.
* **Mutex**: protect device open/close, long I/O sequences, configuration paths.
* **Semaphore**: limit concurrent DMA channels (e.g., max 4 in hardware).
* **RCU**: read-mostly tables (routing entries) with infrequent updates.

---

## 08) “While we use ISR, which technology we use and why?”

* **Top half + Bottom half** split to keep latency low:

  * **Top half (hard IRQ)**: acknowledge hardware, timestamp, queue work, minimal.
  * **Bottom half** via **softirq/tasklet/workqueue/ threaded IRQ**: do heavy lifting (copy buffers, re-arm HW).
* **Why**: hard IRQs run with interrupts masked/preemption off → must be **short**; the rest is deferred.

---

## 09) `malloc` vs `calloc` (user space)

* **malloc(n)**: returns uninitialized memory (whatever was on the heap).
* **calloc(count, size)**: returns **zero-initialized** `count*size`; also checks overflow on multiplication.
* **Performance**: `calloc` may be faster for big zeroed blocks via kernel zero-page tricks, but often similar.

---

## 10) Stack interview Q\&A (quick hits)

* **Stack vs Heap**: automatic vs dynamic storage; LIFO vs free-list.
* **Growth**: typically downward; size limited (`ulimit -s` in user space).
* **Recursion**: each call consumes stack frame (return addr + locals).
* **Reentrancy**: functions using only stack/local state are reentrant.
* **Protection**: guard pages + canaries (user), separate per-thread stacks.

---

## 11) Stack memory of a thread

* **User threads**: default \~8 MB on many distros; set via `pthread_attr_setstacksize()`. Each thread gets its own stack + guard page.
* **Kernel threads**: fixed small stack per task (arch-dependent, commonly 16 KB on x86-64/arm64). Must not allocate big objects on the kernel stack; use heap.

---

## 12) `kmalloc` vs “zmalloc”

* In Linux, it’s **`kmalloc`** vs **`kzalloc`** (zeroed). (“zmalloc” is colloquial.)

  * `kmalloc(size, GFP_KERNEL)` – physically contiguous, cache-aligned, not zeroed.
  * `kzalloc(size, GFP_KERNEL)` – same but zero-initialized.
  * `devm_kzalloc(dev, size, GFP_KERNEL)` – managed lifetime.
  * For large/non-contiguous: `vmalloc()`/`vzalloc()` (virtually contiguous).

---

## 13) Commands to load/unload modules

* **Load**: `modprobe mydrv` (resolves deps) or `insmod mydrv.ko` (no deps).
* **Unload**: `modprobe -r mydrv` or `rmmod mydrv`.
* **Inspect**: `lsmod`, `modinfo mydrv.ko`, `dmesg`.

---

## 14) Static vs Dynamic (module) driver

* **Static (built-in)**: compiled into kernel (`CONFIG_MYDRV=y`).

  * * always available early (e.g., boot storage), − increases kernel image, cannot unload.
* **Dynamic (module)**: `CONFIG_MYDRV=m`.

  * * load on demand, unload for updates/testing, − not usable before initramfs if required for rootfs.

---

## 15) Add nodes in Linux **Device Tree** (DTS) — example

**DTS snippet (enable an I²C sensor):**

```dts
&i2c2 {
    status = "okay";
    temp@48 {
        compatible = "acme,tmp123";
        reg = <0x48>;
        interrupt-parent = <&gpio1>;
        interrupts = <5 IRQ_TYPE_LEVEL_LOW>;
        vdd-supply = <&vdd_3v3>;
    };
};
```

**Driver side:**

```c
static const struct of_device_id tmp123_of_match[] = {
    { .compatible = "acme,tmp123" }, {}
};
MODULE_DEVICE_TABLE(of, tmp123_of_match);
```

**Steps**: edit board `.dts`, include common `.dtsi`, rebuild `dtb`, boot with it; ensure driver has matching `compatible`.

---

## 16) **DTS** vs **DTSI**

* **`.dtsi`**: “include” files with shared SoC/board definitions (reusable).
* **`.dts`**: final board file that includes `.dtsi` and overrides/enables nodes.
  Analogy: `.dtsi` = library; `.dts` = application config.

---

## 17) What is a **structure** (C)

* Aggregate type grouping fields under one name, with **padding** for alignment:

```c
struct packet {
    u16 len;     // 2 bytes
    u8  type;    // 1 byte (+1 pad)
    u32 ts;      // 4 bytes
}; // size likely 8 due to alignment
```

---

## 18) Program to reverse a string (in-place, O(1) space)

```c
// reverse a null-terminated string safely
void str_reverse(char *s) {
    if (!s) return;
    char *l = s, *r = s;
    while (*r) r++;           // find end
    if (r == s) return;
    r--;                      // last char
    while (l < r) {
        char t = *l; *l = *r; *r = t;
        l++; r--;
    }
}
```

---

## 19) Compute **size of a struct** without bitwise tricks (3 methods)

Assume:

```c
struct X { char a; int b; short c; };
```

1. **Pointer arithmetic** (no `sizeof` on struct):

```c
size_t size_of_X(void){
    struct X x;
    return (char*)(&x + 1) - (char*)&x;
}
```

2. **Heap object delta**:

```c
size_t size_of_X_heap(void){
    struct X *p = malloc(sizeof *p);      // only for allocation; not "calculating"
    size_t s = (char*)(p + 1) - (char*)p; // computes struct stride
    free(p); return s;
}
```

3. **Last-member offset + member size** (accounts for padding to last member but misses tail padding unless you round up to alignment of struct):

```c
#include <stddef.h>
size_t approx_size_of_X(void){
    return offsetof(struct X, c) + sizeof(((struct X*)0)->c);
}
```

> Accurate total size (including tail padding) is reliably given by methods (1)/(2). The `offsetof+last` method may **underestimate** if compiler adds tail padding.

---

## 20) Structure padding & how to reduce it

* **Padding** ensures each field’s alignment (e.g., 4-byte aligned `u32`).
* **Mitigation**:

  * Reorder fields from largest → smallest.
  * Use `__packed` only for wire formats/MMIO registers (may hurt performance & cause unaligned accesses).
  * Bundle bitfields or smaller types together.
    **Example reordering** can take 8-byte struct down to 7 or keep alignment tight.

---

## 21) Difference between **insmod** and **probe**

* **`insmod/modprobe`**: user command to load a module image into the kernel.
* **`.probe()`**: driver callback invoked by the **bus core** when a device **matches** the driver (via DT `compatible`, ACPI ID, PCI ID, etc.).
  You can load a module and **never** get `probe()` if no device matches.

---

## 22) What is **IOCTL**

* **ioctl(fd, cmd, arg)** user-space syscall for device-specific control operations that don’t fit read/write.
* In kernel char driver: implement `.unlocked_ioctl = my_ioctl` (and optionally `.compat_ioctl` for 32-bit).
* **Command encoding** via `_IO, _IOR, _IOW, _IOWR(type, nr, struct)` defines direction and size for copy checks.

---

## 23) Where do we use IOCTL? Real scenarios

* Set SPI mode/bitrate, I²C timeouts.
* Configure UART termios (e.g., `TIOCSSERIAL`), modem control.
* NIC ethtool ops (`SIOCETHTOOL`), PHY settings.
* Custom device: set sampling rate, read sensor calibration blob, map user buffers for DMA.

**Example header:**

```c
#define MYDEV_IOC_MAGIC 'M'
#define MYDEV_SET_RATE  _IOW(MYDEV_IOC_MAGIC, 1, unsigned int)
#define MYDEV_GET_STAT  _IOR(MYDEV_IOC_MAGIC, 2, struct my_stats)
```

---

## 24) What to modify when adding a **new driver** to kernel source

* **Kconfig**: add a menu entry (`config MYDRV …`).
* **Makefile**: add `obj-$(CONFIG_MYDRV) += mydrv.o`.
* **Bindings** (DT): add YAML under `Documentation/devicetree/bindings/...` and run `dtbs_check`.
* **Defconfig**: enable `CONFIG_MYDRV` for your build.
* **DTS**: add node with matching `compatible`.
* **MAINTAINERS**: optional but good hygiene.

---

## 25) What is **sysfs**?

* Virtual FS at `/sys` exposing **kernel object model** (`kobject`/`kset`) as attributes.
* One file per attribute; text interface; udev uses it to create `/dev/*`.
* Drivers create groups via `sysfs_create_group()` or use `device_create_file()`; in modern code prefer **`device_attribute`** and **`dev_groups`** in `struct device`.

---

## 26) Major & Minor numbers

* **Major** identifies the driver (the “class”).
* **Minor** identifies an instance (device) handled by that driver.
* Packed in `dev_t`. Examples: `/dev/ttyS0` (major 4), `/dev/sda` (block major 8).

---

## 27) **sysfs** vs **proc**

* **/proc**: process & kernel runtime info (task status, memory, mounts). Historical dumping ground.
* **/sys**: structured attributes for devices, drivers, buses (hotplug, power, uevents).
  Rule: device/driver configuration → **sysfs**; process info/statistics → **proc**.

---

## 28) API/macros to get major/minor

* From `dev_t`: `MAJOR(dev_t)`, `MINOR(dev_t)`, `MKDEV(major, minor)`.
* From an inode in file ops: `imajor(inode)`, `iminor(inode)`.

---

## 29) File operations in a driver (userspace ↔ kernel flow)

**Userspace**: `fd = open("/dev/mydev", O_RDWR); read(fd,…); write(fd,…); ioctl(fd,…); close(fd);`
**Kernel**:

* VFS resolves `/dev/mydev` → `inode` → `cdev` → your `file_operations`.
* Typical skeleton:

```c
static ssize_t my_read(struct file *f, char __user *ubuf, size_t len, loff_t *off) {
    size_t n = min(len, (size_t)avail);
    if (copy_to_user(ubuf, kbuf, n)) return -EFAULT;
    return n;
}
static const struct file_operations my_fops = {
    .owner = THIS_MODULE,
    .read = my_read, .write = my_write,
    .open = my_open, .release = my_release,
    .unlocked_ioctl = my_ioctl,
};
```

**Data path**: VFS → your `.read/.write/.ioctl` → driver/HW.

---

## 30) What is the **stack section**?

In a process’s virtual memory layout: **text/code**, **rodata**, **data**, **bss**, **heap (brk/mmap)**, **stack** (grows downward). The “stack section” is the memory region dedicated to function call frames (return addr, saved regs, locals).

---

## 31) How variables get memory; how to access heap

* **Static storage** (`static`/globals): in `.data` (initialized) or `.bss` (zeroed) for the program’s lifetime.
* **Automatic storage** (locals): on the **stack**; lifetime = scope/call.
* **Dynamic** (**heap**): `malloc/calloc/realloc/free` (user) or `kmalloc/kzalloc/vmalloc/kfree` (kernel).
* **Access heap**: keep pointers returned by allocator; free with matching API; in kernel choose proper `GFP_*` flags (`GFP_KERNEL`, `GFP_ATOMIC` in IRQ).

---

## 32) `const` vs `volatile` (+ what if volatile missing)

* **`const`**: read-only from the perspective of your code; may end up in `.rodata`. Doesn’t affect hardware accesses.
* **`volatile`**: tells compiler **do not optimize away** loads/stores because the value may change outside program control (MMIO regs, shared with ISR).
* **If you omit `volatile`** on a polled HW status register or a flag set in an ISR, the compiler might cache the value → infinite loop or missed events.
* **Kernel best practice**:

  * For **MMIO**, don’t rely on `volatile`; use accessors `readl/writel`, `ioread8/16/32` (they imply barriers).
  * For shared vars, use `READ_ONCE/WRITE_ONCE`, plus proper **memory barriers** or locks.

---

## 33) What is a **segmentation fault**?

* User-space crash when accessing memory not mapped/allowed (bad pointer, execute non-exec page). Kernel sees a page fault; if unhandled in user context → SIGSEGV to the process.

---

## 34) What is a **kernel panic**?

* Fatal, unrecoverable kernel error (e.g., BUG(), null deref in kernel, corrupt list, inconsistent locks). System halts/reboots depending on `panic=` parameter. Produces stack trace in `dmesg`.

---

## 35) What is a **stack overflow**?

* Exceeding thread stack limit: deep recursion or large local arrays.
  User: SIGSEGV on guard page. Kernel: can corrupt state → often panic. Avoid large on-stack buffers; use heap.

---

## 36) Copy data **user↔kernel** APIs

* **Kernel → user**: `copy_to_user(void __user *to, const void *from, unsigned long n)`
* **User → kernel**: `copy_from_user(void *to, const void __user *from, unsigned long n)`
* Return value = **bytes not copied** (0 on success). Check and return `-EFAULT` if non-zero.

---

## 37) Other related APIs (besides copy\_to/from\_user)

* **Single values**: `get_user(x, __user *p)`, `put_user(x, __user *p)`
* **Strings**: `strnlen_user(__user *s, long n)`, `strncpy_from_user(char *dst, __user *src, long n)`
* **User↔user within kernel**: `copy_in_user()` (rare).
* **Struct-aware**: `copy_struct_from_user()` (helps with versioned/flexible structs).
* **I/O memory** (not user copy, but often confused): `memcpy_fromio()/memcpy_toio()`, `readl/writel`.

---

### Bonus: Minimal IRQ skeleton (with bottom half)

```c
// concise comments per your style
static irqreturn_t my_irq_top(int irq, void *dev_id) {
    struct mydev *d = dev_id;
    u32 status = readl(d->base + STAT);
    if (!(status & INTR_BIT)) return IRQ_NONE;      // spurious
    writel(INTR_BIT, d->base + STAT);               // ack
    // queue bottom-half to process data
    schedule_work(&d->wq);
    return IRQ_HANDLED;
}

static void my_work(struct work_struct *w) {
    struct mydev *d = container_of(w, struct mydev, wq);
    // safe to sleep; do DMA mapping/copies/etc.
}

ret = request_irq(d->irq, my_irq_top, IRQF_TRIGGER_RISING, "mydev", d);
INIT_WORK(&d->wq, my_work);
```

If you want, I can turn any section into a printable cheat-sheet or add a tiny sample char driver with `open/read/write/ioctl` + DT matching in your preferred style.
