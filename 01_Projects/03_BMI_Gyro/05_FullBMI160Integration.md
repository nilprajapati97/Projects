Great! Let's walk through the full BMI160 sensor integration flow on SDM660 using SLPI and Qualcomm’s sensor framework with Hexagon SDK and SDF.

---

## ✅ Overview

**Goal:** Enable BMI160 on SDM660 via SLPI (Sensor Low Power Island), so sensor data is processed efficiently without waking up the Application Processor (AP).

---

## 🧠 Qualcomm Sensor Framework – Architecture

```
+---------------------------+
|     Android HAL          | ← Uses sensors HAL (via libsensor1 or AIDL)
+---------------------------+
|     SSC Driver (SNS)     | ← Communicates with SSC on SLPI via FastRPC
+---------------------------+
|     FastRPC Channel      | ← Marshals messages to/from SLPI
+---------------------------+
|     SLPI Sensor Core     |
| +----------------------+ |
| |  Sensor Manager (SMGR)| |
| |  Framework (SDF)      | |
| |  BMI160 Driver (Sensor)| |
| +----------------------+ |
+---------------------------+
```

---

## 🔌 Hardware Connection (from your Device Tree)

```dts
&i2c_3 {
    bmi160@68 {
        compatible = "bosch,bmi160";
        reg = <0x68>;
        interrupt-parent = <&tlmm>;
        interrupts = <23 IRQ_TYPE_EDGE_RISING>;
        vdd-supply = <&vreg_l7>;
        vddio-supply = <&vreg_l5>;
    };
};
```

* **I2C address:** `0x68` (standard for BMI160)
* **IRQ:** GPIO 23 → interrupt from sensor (data ready)
* **Power Rails:** L7 (core) and L5 (IO)

---

## 🧩 SLPI Side - Integration Steps

### 1. **Hexagon SDK Setup**

* Use `Hexagon_SDK_X.Y.Z/` with `SLPI_PROC` as your target processor.
* Build the driver as part of the `sensors` module.

### 2. **BMI160 Sensor Driver (SLPI Side)**

* Located in `sensors/dri/bosch/bmi160/`
* Main components:

  * `sns_bmi160_sensor.c` → Registers sensor with Sensor Manager.
  * `sns_bmi160_sensor_instance.c` → Handles runtime logic.
  * `sns_bmi160_hal.c` → Abstracts I2C reads/writes.
  * `sns_bmi160_reg.c` → Sensor register defs.

---

### 3. **SDF (Sensor Driver Framework)**

Defines how sensor drivers register, initialize, and communicate with the Sensor Manager.

* You define the driver in the `sns_bmi160_sensor_api` structure.
* It registers itself with the `Sensor Manager`, advertising capabilities.

```c
sns_sensor_api sns_bmi160_sensor_api = {
    .struct_len = sizeof(sns_sensor_api),
    .init = &sns_bmi160_init,
    .deinit = &sns_bmi160_deinit,
    .get_sensor_uid = &sns_bmi160_get_sensor_uid,
    .set_client_request = &sns_bmi160_set_client_request,
    .notify_event = &sns_bmi160_notify_event,
};
```

---

## 🛠️ Data Flow at Runtime

1. **Host requests sensor data** via sensors HAL
2. **Sensor1 client library** sends FastRPC request to SLPI
3. **SLPI SMGR** sends activation message to `bmi160` instance
4. **BMI160 driver** sets up interrupts, FIFO, I2C polling
5. Sensor starts collecting data and fires **GPIO interrupt**
6. **SLPI driver** reads FIFO via I2C
7. Parsed data sent back to AP via **indications (sns\_client\_event)** through FastRPC

---

## 🔄 Sensor Client Use (on Android)

```c
sensor1_write(client_handle, &request_msg, ...);
// or via HAL wrapper (AIDL in newer Androids)
```

---

## 🔧 Build + Flash (High Level)

```bash
# Set environment
source setenv.sh
make sensors_proc

# Output: hexagon/Sensor/dynamic/bmi160*.elf
```

* Output goes to `slpi_proc/build/`
* Flash onto SDM660 board via QFIL or fastboot depending on your setup.

---

Would you like the step-by-step template for writing a custom driver (e.g., starting from sns\_bmi160\_sensor.c) or FastRPC `.idl` + client/server templates next?
