SLPI
========================================================================================================
SLPI (Sensor Low Power Island) is a low-power DSP core used in Qualcomm SoCs (like SDM660) to offload sensor data processing from the main application processor (AP), improving power efficiency and performance.

📦 Where SLPI fits in SDM660 Architecture
=========================================

                 +------------------------+
                 |    Application CPU     |
                 |    (Snapdragon A53)    |
                 +------------------------+
                           |
                           | FastRPC / HLOS Driver
                           |
        +------------------▼------------------+
        |               SLPI DSP              |
        | (QDSP6/Hexagon core inside SDM660)  |
        +------------------▲------------------+
                           |
                           | Sensors (I2C/SPI/PMIC)
                           ▼
                  Accelerometer, Gyro, etc.

⚙️ SLPI Architecture on SDM660
======================================
SLPI is one of several DSPs in Qualcomm SoCs:
    ADSP (Audio DSP) – handles audio
    CDSP (Compute DSP) – handles ML/vision
    SLPI (Sensor DSP) – handles IMU sensors (accel, gyro, etc.)

The Hexagon DSP core inside SLPI executes sensor-related algorithms and real-time processing (e.g., motion detection, step counting) independently of the main CPU.

🔄 Data Flow – Without vs With SLPI
===========================================================================
❌ Without SLPI:
    Sensors communicate directly with Application Processor (AP)
    All processing (filtering, fusion) happens on the CPU
    High power consumption since AP is constantly woken

✅ With SLPI Offloading:
===========================================================================
1. Sensor signals go to the SLPI via SPI/I2C (usually via PMIC or sensor hub).
2. SLPI executes code to handle raw sensor data (like filtering, fusion, pedometer, gestures).
3. SLPI sends only processed events to the AP.
4. Communication is via FastRPC mechanism.

🛠️ FastRPC – The Bridge Between CPU and SLPI
===========================================================================
FastRPC is Qualcomm's IPC mechanism for HLOS (High-Level OS like Android/Linux) to communicate with DSPs (like SLPI/ADSP).

🧩 Components Involved:
----------------------------------------------------------------------------
Shared memory buffer: Between SLPI and HLOS.
RPC stubs and skeletons: Code generated via .idl files for both ends.
Dynamic shared object (.so) files:
On SLPI: compiled into .elf and loaded into DSP
On HLOS: uses libadsprpc.so or libsensor1.so to call into DSP

🔧 SLPI Offloading Example:
------------------------------
Let’s say you're offloading a pedometer algorithm:
sensor_pedometer.c runs on SLPI.
It collects accelerometer data, applies filtering, counts steps.
Uses FastRPC to notify HLOS: "X steps counted."
The Android HAL (Sensor HAL) reads this data via sensor1 API or sns_low_lat framework.

🔒 Why Offload to SLPI?
--------------------------------------------------------------
Power efficiency: SLPI can run while AP is in suspend.
Real-time processing: SLPI has predictable latency.
Modularity: offloaded algorithms can be updated independently via .mbn firmware.




