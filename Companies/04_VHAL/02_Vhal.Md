🔹 1. Java App Layer (Car Apps)
======================================================================================
Android Automotive apps use CarPropertyManager to interact with vehicle info.


CarPropertyManager carPropMgr = car.createCarPropertyManager();
carPropMgr.getProperty(...); // speed, RPM, HVAC


🔹 2. Car Service (Java Framework Layer)
======================================================================================
Lives in packages/services/Car

It abstracts the vehicle features into proper APIs.

Talks to Vehicle HAL via JNI.

Example:
CarPowerManagementService, CarSensorService, etc.

🔹 3. JNI Layer: VehicleHalManager.cpp
======================================================================================
Bridges Java and native code:

Loads the AIDL Vehicle HAL service

Translates property requests to C++ calls


android::sp<IVehicle> mVehicle;
mVehicle->get(...);  // Calls into HAL
🔹 4. Vehicle HAL AIDL Interface
======================================================================================
Defined in:
hardware/interfaces/automotive/vehicle/aidl/android/hardware/automotive/vehicle/IVehicle.aidl

Key Functions:

IVehicle {
    oneway void setValue(VehiclePropValue value);
    VehiclePropValue getValue(int32 propId);
    ...
}
It defines vehicle property IDs, status, access type, and more.

🔹 5. Vehicle HAL C++ Implementation
======================================================================================
Code:
`hardware/interfaces/automotive/vehicle/aidl/default/DefaultVehicleHal.cpp`

Responsibilities:

Maintains property state in memory or fetches via hardware.

Maps prop IDs like:
    VehicleProperty::CURRENT_GEAR
    VehicleProperty::HVAC_FAN_SPEED
    VehicleProperty::ENGINE_RPM

Uses VehiclePropValue structure to encode values.

Sample:

VehiclePropValue DefaultVehicleHal::getValue(...) {
    switch (propId) {
        case CURRENT_GEAR:
            return constructGearProp(3); // Drive
        ...
    }
}


🔹 6. Hardware Interface (CAN / Serial / BMS)
======================================================================================
Interacts with hardware using:

CAN bus (via SocketCAN or MCP2515)

UART (MCU-to-Android)

I2C (BMS or sensors)

Data Parsing Layer:

May use custom protocol to read vehicle messages.

Implements message framing, checksum, etc.

Example:


int canSocket = socket(PF_CAN, SOCK_RAW, CAN_RAW);
bind(canSocket, ...);
read(canSocket, &frame, sizeof(frame));
parseCAN(frame);

🔹 7. Device Driver Layer (Optional)
======================================================================================
Some systems may use character device drivers: /dev/vehicle

HAL can open this and communicate with MCU directly.


int fd = open("/dev/vehicle", O_RDWR);
write(fd, command, len);
read(fd, response, len);

🔐 8. SELinux + Service Registration
======================================================================================
AIDL HAL service is registered as:

    android.hardware.automotive.vehicle.IVehicle/default

Must be listed in:

- `vintf/manifest.xml`
- rc file to start with system:

    service vehicle-hal /vendor/bin/hw/android.hardware.automotive.vehicle-service

SELinux rules:

- Define `hal_vehicle_default` context.
- Allow access to `/dev/can0`, `/dev/vehicle`, etc.

💡 Example Flow: App Reads Vehicle Speed
======================================================================================

- Java App  
  ↓  
- CarPropertyManager.get(SPEED)  
  ↓  
- CarSensorManager → JNI  
  ↓  
- IVehicle.get(SPEED)  
  ↓  
- Vehicle HAL C++ → parses cache or hardware  
  ↓  
- Returns `VehiclePropValue { propId: SPEED, value: 45.0 }`  
  ↓  
- App displays speed


🔁 Periodic Update or Subscription
======================================================================================
Vehicle HAL supports subscriptions:

Framework subscribes to a prop (like speed).

HAL pushes updates via `IVehicleCallback`:

📋 Common VehicleProp IDs
======================================================================================

| Prop ID              | Type    | Example Use           |
|----------------------|---------|-----------------------|
| PERF_VEHICLE_SPEED   | float   | Car speed (km/h)      |
| HVAC_FAN_SPEED       | int     | Fan speed             |
| GEAR_SELECTION       | int     | PRND gear info        |
| DOOR_POSITIONS       | int[]   | Door open/close       |
| IGNITION_STATE       | enum    | OFF/ACC/RUN           |

See:
android.hardware.automotive.vehicle@2.0/types.hal

✅ Summary (Block-wise)
======================================================================================

Apps (Java)  →  CarService  →  JNI  →  IVehicle AIDL  →  HAL Impl (C++)  →  ECU / Driver
      ↑             ↑             ↑            ↑                ↑
  Subscription   Subscription   Subscription   get/set   Hardware communication (CAN/UART)



🔧 Debugging Tools
======================================================================================

| Tool             | Use                                      |
|------------------|------------------------------------------|
| lshal            | List HALs and their status               |
| adb logcat       | Framework + HAL logs                     |
| dumpsys car      | Check car property manager, speed, HVAC  |
| vhal_test_client | Test vehicle HAL APIs                    |