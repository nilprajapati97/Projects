Design and develop solutions for new products and enhancements for existing
products
==================================================================================

🧠 Section 1: System Design & Architecture
-----------------------------------------------------------------------------------
🔸 Q1: How would you design an embedded Linux system for a new hardware product?
Answer:

+ Start with requirements: latency, boot time, memory, power.
+ Choose SoC and board support package (BSP).
+ Select Yocto/OpenWRT/Buildroot for Linux distro.
+ Define bootloader (e.g. U-Boot) + partitioning (boot, rootfs, data).
+ Customize kernel: enable drivers, I2C/SPI/UART, power management.
+ Write custom device trees for hardware components.
+ Design secure OTA mechanism.
+ Ensure security (secure boot, trusted firmware, dm-verity, SELinux).
+ Plan production programming and factory test flows.

🔸 Q2: How do you isolate bugs in a newly developed board with a Linux OS?
Answer:

- Check hardware signals using oscilloscope/logic analyzer.
- Use `dmesg`, `earlyprintk`, `initcall_debug` for early boot issues.
- Validate U-Boot output over UART.
- Test DDR via stress test or `memtester`.
- Check device tree pinmux and regulator settings.
- Use static IP and `ping` to test network.
- Enable `CONFIG_DEBUG_LL` and `CONFIG_EARLY_PRINTK`.

🧠 Section 2: Linux Device Drivers
🔸 Q3: Explain how you would write a Linux device driver for a GPIO-controlled buzzer.
Answer:

-  Request GPIO using gpiod_get() or gpio_request() (legacy).
-  Set GPIO direction to output.
-  Expose sysfs entry (or use miscdevice) for user control.
-  In write(), toggle GPIO on/off.
-  Add device tree binding in .dts.
🔧 Bonus: Use devm_ APIs to ensure proper cleanup.

🔸 Q4: What’s the difference between a platform driver and a character driver?
Answer:

-  Platform driver is tied to devices declared in device tree (non-discoverable hardware).
-  Character driver is a type of interface (e.g., read/write to /dev/buzzer).
-  Platform drivers usually register character devices inside probe function.
// Example code for a platform driver registering a character device
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/uaccess.h>
platform_driver_register(&my_driver); // matches DT
cdev_add(&cdev, dev_num, 1);          // registers /dev interface

🔸 Q5: How do you handle concurrency in a driver?
Answer:

- Use spinlocks for atomic context (ISR-safe).
- Use mutexes for user-space APIs.
- Avoid sleeping in interrupt context.
- Protect shared buffers between ISR and user context.

🧠 Section 3: Kernel, Bootloader, & BSP
🔸 Q6: What is the boot sequence from power-on to shell in embedded Linux?
Answer:

- Power-on/reset
- SoC ROM loader (boot ROM)
- Bootloader stage 1 (SPL/FSBL)
- Bootloader stage 2 (U-Boot)
- Load and decompress kernel
- Load DTB
- Mount root filesystem
- Start init (systemd/init)
- User services and shell/login

🔸 Q7: How do you reduce boot time in embedded Linux?
Answer:

- Optimize kernel configuration (remove unused features).
- Use initramfs instead of full rootfs.
- Parallelize init with systemd.
- Optimize U-Boot delay, remove console output.
- Compress kernel with LZ4 instead of gzip.
- Use Device Tree overlays to avoid runtime probing.
- Pre-initialize peripherals where safe.

🔸 Q8: How do you upgrade firmware securely in the field?
Answer:

- Use secure boot to verify bootloader and kernel.
- Implement OTA updates with signed firmware images.
- Use dual partition scheme (A/B rootfs) to allow rollback.
- Validate firmware signatures using public key.
- Use dm-verity to ensure rootfs integrity.
- Use mtd-utils, fw_setenv, or secure OTA clients (e.g., RAUC, SWUpdate, Mender).
- Implement rollback mechanism if boot fails.   
- Use secure storage for keys and signatures.

🧠 Section 4: Debugging & Tools
🔸 Q9: How do you debug a kernel crash?
Answer:
- Enable kernel crash dump (kdump) to capture memory on panic.
- Use `CONFIG_KGDB` to connect gdb over serial.
- Analyze panic logs via UART (e.g., `dmesg`, `ftrace`, or `perf`).
- Use `addr2line` on vmlinux to resolve addresses.
- Check Oops messages for backtrace and register state.
- Use `gdb` to analyze core dumps if available.
- Use `ftrace` to trace function calls leading to the crash.

🔸 Q10: What is strace and when do you use it?
Answer:

- strace traces system calls made by a process.
- Useful for debugging:
- File not found (open())
- Permission issues (access())
- IPC and signals (read()/write())

strace -f -o trace.log ./my_app

🧠 Section 5: Build System / Yocto
🔸 Q11: How do you add a new driver into the Yocto build?
Answer:
- Create a recipe (.bb) for your driver.
- Add your driver source code to the recipe.
- Specify dependencies in the recipe (e.g., kernel version).
- Add your recipe to the appropriate layer (e.g., meta-custom).
- Add your recipe to `IMAGE_INSTALL` in your image recipe.
- If needed, add to `KERNEL_DEVICETREE` for device tree support.

🔸 Q12: How do you use devicetree to enable an I2C peripheral?
Answer:
- Define the I2C bus in the device tree source (.dts) file.
- Specify the I2C device node with compatible properties.
- Include the device tree overlay if needed.
- Rebuild the device tree binary (.dtb) and deploy it.
- Load the new device tree on boot (e.g., via U-Boot).

&i2c2 {
    my_sensor@30 {
        compatible = "vendor,my-sensor";
        reg = <0x30>;
        interrupt-parent = <&gpio1>;
        interrupts = <12 IRQ_TYPE_EDGE_FALLING>;
    };
};


🧠 Section 6: Soft Skills & Ownership
🔸 Q13: Tell me about a product you helped bring from design to production.
Answer (Sample):
"I was involved from schematic review to kernel driver development for a thermal sensor. 
I wrote the I2C driver, verified timing with a logic analyzer, and built a HAL exposed via sysfs.
I also helped define the OTA mechanism using A/B partitions and contributed test cases for 
manufacturing QA scripts."

🔸 Q14: How do you ensure code quality and reliability in embedded systems?
Answer:
- Follow coding standards (e.g., MISRA, Linux kernel style).
- Use static analysis tools (cppcheck, clang-tidy).
- Write unit tests (CMocka, Unity).
- Perform integration tests with hardware.
- Use ftrace to validate timing and performance.
- Automate builds with CI/CD (Jenkins + Yocto).
- Perform long soak and stress testing with watchdogs.

🧰 Tools You Should Be Comfortable With

| Category     | Tools                          |
| ------------ | ------------------------------ |
| Debugging    | gdb, kgdb, perf, ftrace, kdump |
| Kernel Dev   | dmesg, insmod, rmmod, modinfo  |
| Flashing     | fastboot, dfu-util, uuu        |
| Profiling    | valgrind, cachegrind, perf     |
| Build System | Yocto, BitBake, CMake          |

✅ Summary Areas to Prepare
--------------------------------
| Area                  | Focus                          |
| --------------------- | ------------------------------ |
| Driver Development    | Platform drivers, char devices |
| Linux Kernel Concepts | Boot flow, memory, concurrency |
| Embedded Debugging    | UART logs, crash analysis      |
| Security/OTA          | Verified boot, dm-verity, A/B  |
| Build/Packaging       | Yocto, kernel config, u-boot   |

