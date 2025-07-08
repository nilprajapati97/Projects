✅ Project Summary
======================================================================================================
I was responsible for the end-to-end integration of a GNSS (GPS) module, embedded within a WWAN modem, into a custom Linux-based embedded platform. The goal was to provide standardized location services to the system using ModemManager, with NMEA sentence parsing, without writing any custom HAL or GPS service.

Use Case -:
==============
This was part of a production-grade embedded system, targeting use-cases like vehicle tracking, secure payments, and location-based access control.

🔍 Project Context and Motivation
======================================================================================================
1. In modern devices, GNSS support is critical — for time sync, location tracking, security
     enforcement, or regulatory compliance (like eCall in EU).
2. We had a multi-function WWAN modem (e.g., Quectel EC25 or Telit LE910) which provided GNSS over a
    serial port — no need for a discrete GPS chip.
3. The customer requirement was to avoid proprietary stack, reuse Linux-native services like 
    ModemManager and D-Bus, and ensure system-level visibility and diagnostics.

🧩 Key Components and Technologies Used
======================================================================================================
| Component            | Role                                                                |
| -------------------- | ------------------------------------------------------------------- |
| **ModemManager**     | Manages modem and GPS via high-level D-Bus APIs                     |
| **NMEA Protocol**    | Standard format used by GPS to report location data                 |
| **/dev/ttyUSBx**     | Serial port used to stream GNSS data                                |
| **mmcli Tool**       | Command-line interface for modem control and testing                |
| **Lauterbach / PLS** | For deep driver-level and register-level debug                      |
| **Linux BSP**        | Custom Yocto-based BSP with USB, UART, PM, and modem driver support |

🔧 Responsibilities and Execution
======================================================================================================

📌 1. Low-Level Hardware Setup
--------------------------------
1. Brought up USB ports, configured power domains, and ensured modem detected via usbserial, option.c.
2. Verified /dev/ttyUSB0–4 enumeration, including GNSS interface

