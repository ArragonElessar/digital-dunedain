---
title: "Building a Real-Time IMU Data Broadcaster with Zephyr RTOS and BLE"
date: 2026-04-21T10:00:00-05:00
draft: false
tags: ["Zephyr", "RTOS", "BLE", "IoT", "Embedded", "C", "Sensors", "IMU"]
categories: ["Embedded Systems"]
---

In the world of IoT and connected devices, streaming real-time sensor data efficiently is a common challenge. How do you get motion data from a device to a nearby smartphone or gateway without the overhead of a full connection? This project provides a practical answer by building an Inertial Measurement Unit (IMU) data broadcaster using the powerful [Zephyr RTOS](https://www.zephyrproject.org/) and Bluetooth Low Energy (BLE).

We'll take a deep dive into a project that reads accelerometer and gyroscope data from an LSM6DSO sensor and broadcasts it wirelessly in BLE advertising packets. This allows any BLE-capable device in the vicinity to listen in on the data stream, no connection required!

## The System Architecture: A Bird's-Eye View

Before we get into the code, let's understand the main components and how they interact:

1.  **The Sensor (LSM6DSO IMU)**: This is our data source. An IMU like the [LSM6DSO from STMicroelectronics](https://www.st.com/en/mems-and-sensors/lsm6dso.html) combines an accelerometer and a gyroscope to measure both linear acceleration and angular velocity across three axes.
2.  **The Communication Bus (I²C)**: This is the electrical pathway between our microcontroller and the IMU. [I²C (Inter-Integrated Circuit)](https://learn.sparkfun.com/tutorials/i2c/all) is a popular two-wire serial protocol perfect for connecting peripherals like sensors to a processor.
3.  **The Brain (Microcontroller running Zephyr RTOS)**: The MCU orchestrates everything. It runs the Zephyr RTOS, which provides the necessary drivers (I²C, BLE) and kernel features (timing, scheduling) to make our application robust and reliable.
4.  **The Wireless Link (Bluetooth Low Energy)**: We use BLE not in its typical connected mode, but as a simple broadcaster. The MCU packages the sensor data into custom advertising packets and sends them out periodically.
5.  **The Listener (Any BLE Scanner)**: A smartphone, laptop, or another microcontroller can scan for these advertising packets, parse the data, and use it for applications like motion tracking, orientation visualization, or gesture recognition.

![System Diagram](https://i.imgur.com/your-diagram-link.png) <!-- You would replace this with a link to an actual diagram -->

## Software Deep Dive: Inside the Zephyr Application

The magic happens in the software. Zephyr's structured environment helps us separate hardware configuration from application logic.
### [Github Link](https://github.com/ArragonElessar/nrf54l15_imu_ble)

### 1. Describing the Hardware: The Device Tree

Zephyr uses the [Device Tree](https://docs.zephyrproject.org/latest/build/dts/index.html) to describe the hardware layout of a board in a source file (`.dts` or `.overlay`). This abstracts the hardware, making our application code more portable.

Our code needs to know which I²C bus the sensor is on. We use a nodelabel, `i2c30`, to get a device pointer.

```c
// From src/main.c
const struct device *i2c_dev = DEVICE_DT_GET(DT_NODELABEL(i2c30));
```

For a board like the `nrf52840dk_nrf52840`, you would create an `nrf52840dk_nrf52840.overlay` file to define this:

```devicetree
/* Example .overlay file */
&i2c0 {
    status = "okay";
    sda-pin = <26>;
    scl-pin = <27>;

    lsm6dso@6a {
        compatible = "st,lsm6dso";
        reg = <0x6a>;
        label = "LSM6DSO";
    };
};

/* We create a nodelabel for our code to find i2c0 */
/ {
    aliases {
        i2c30 = &i2c0;
    };
};
```

### 2. The Sensor Driver (`lsm6dso.c`)

This file contains the low-level functions for interacting with the IMU.

#### Initialization (`lsm6dso_setup`)

The setup function configures the sensor for our needs.

1.  **Sanity Check**: It first reads the `WHO_AM_I` register. This is a fixed-value register on the sensor. If the value read back matches the expected value (`0x6A`), we can be confident our I²C communication is working.
2.  **Configure Accelerometer**: We write to `REG_CTRL1_XL` to set the accelerometer's Output Data Rate (ODR) and full-scale range. In this project, we set it to 104 Hz and ±2g. This means it will produce 104 new readings per second, and the data will represent acceleration values up to twice the force of gravity.
3.  **Configure Gyroscope**: Similarly, we write to `REG_CTRL2_G` to set the gyroscope's ODR to 104 Hz and its full-scale range to ±250 degrees per second (dps).

#### Data Fetching (`lsm6dso_fetch_raw_data`)

This function reads 12 consecutive bytes from the sensor starting from the gyroscope output registers. This is called a "burst read" and is much more efficient than reading each of the 12 bytes individually.

The data is "raw" because it's in the form of 16-bit signed integers (from -32768 to 32767). This raw data needs to be converted into meaningful physical units.

#### Data Conversion (`print_lsm6dso_data`)

To convert the raw integer values into g's (for acceleration) and dps (for angular velocity), we use a sensitivity factor provided by the sensor's datasheet.

*   `Accel_g = raw_accel * LSM6DSO_ACCEL_SENSITIVITY_2G`
*   `Gyro_dps = raw_gyro * LSM6DSO_GYRO_SENSITIVITY_250DPS`

```c
// From lsm6dso.h
#define LSM6DSO_ACCEL_SENSITIVITY_2G    0.000598f // mg / LSB
#define LSM6DSO_GYRO_SENSITIVITY_250DPS 0.00762f  // dps / LSB
```

> **Note:** The accelerometer sensitivity is often given in **mg/LSB** (milli-g per Least Significant Bit). Our code converts it to **g/LSB** for the final calculation.

### 3. The BLE Broadcaster (`main.c`)

This is the heart of our application, bringing the sensor data to the wireless world.

#### Broadcasting vs. Connecting

BLE has two primary ways of communicating:
*   **Connected Mode**: A Central (like a phone) establishes a persistent, two-way connection with a Peripheral (our device). This is great for complex interactions but has connection setup overhead and higher power consumption.
*   **Broadcasting/Observing**: A Broadcaster (our device) simply sends out data in small packets. An Observer (a scanner) can listen for these packets without any connection. This is extremely power-efficient and perfect for one-way data streaming.

Our project uses the broadcasting model. For a great primer on BLE advertising, check out this article from Novel Bits.

#### Crafting the Advertising Packet

The content of our broadcast is defined in an array of `bt_data` structs.

```c
// From src/main.c
static const struct bt_data ad[] = {
    BT_DATA(BT_DATA_FLAGS, BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR, sizeof(uint8_t)),
    BT_DATA(BT_DATA_NAME_COMPLETE, "IMU", 3),
    BT_DATA(BT_DATA_MANUFACTURER_DATA, ble_data, sizeof(ble_data))
};
```

Let's break this down:
1.  **Flags**: `BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR` tells scanners that this is a general discoverable device and it does not support Basic Rate/Enhanced Data Rate (i.e., it's BLE-only).
2.  **Complete Name**: We give our device the friendly name "IMU".
3.  **Manufacturer Specific Data**: This is where we put our custom payload! It's a flexible field for vendors to put whatever they want. We define a payload structure:
    *   **Company ID (2 bytes)**: We use a custom ID `0xABFF`. A real product would use an officially assigned ID.
    *   **Sensor Data (12 bytes)**: The raw, packed `lsm6dso_data` struct containing 6 `int16_t` values (3 for accel, 3 for gyro).

#### The Main Loop: Orchestrating the Flow

The `main` function's `while(1)` loop is the engine that drives the application.

```c
while(1)
{
    // 1. Get the start time
    start_time_ms = k_uptime_get();

    // 2. Fetch raw data from the sensor
    lsm6dso_fetch_raw_data(i2c_dev, &data);

    // 3. Print human-readable data to the console
    print_lsm6dso_data(&data);

    // 4. Prepare the BLE payload
    ble_data = 0xff; // Company ID LSB
    ble_data = 0xAB; // Company ID MSB
    memcpy(&ble_data, &data, sizeof(data));

    // 5. Update the advertising data for the next broadcast
    bt_le_adv_update_data(ad, ARRAY_SIZE(ad), NULL, 0);

    // 6. Calculate sleep time to maintain a constant sampling rate
    processing_time_ms = k_uptime_get() - start_time_ms;
    sleep_time_ms = SAMPLING_INTERVAL_MS - processing_time_ms;
    k_msleep(sleep_time_ms);
}
```

The timing logic at the end is crucial for a real-time system. It ensures that we sample at a consistent rate (2 Hz in this case) by accounting for the time it took to fetch, print, and process the data in each loop iteration.

### 4. Receiving the Data: A Simple Python Listener

On the other end, we can use a simple Python script with the excellent Bleak library to receive and parse the data.

The script scans for a device named "IMU". When it finds it, it extracts the manufacturer data. The key is using `struct.unpack` with the correct format string (`'<hhhhhh'`) to convert the 12 bytes of payload back into six 16-bit signed integers.

```python
# From LinuxBLE/main.py
import struct
from bleak import BleakScanner

async def detection_callback(device, advertisement_data):
    if device.name == "IMU":
        # The key is advertisement_data.manufacturer_data
        for company_id, payload in advertisement_data.manufacturer_data.items():
            # '<' for little-endian, 'h' for signed short (16-bit)
            data = struct.unpack('<hhhhhh', payload)
            print(f"Acc: X={data} Y={data} Z={data}")
            print(f"Gyro: X={data} Y={data} Z={data}")

# ... main async loop ...
```

## Conclusion and Next Steps

This project serves as a robust template for building simple, efficient, one-way sensor data streams with Zephyr and BLE. By leveraging manufacturer data in advertising packets, we avoid the complexity and power cost of a full BLE connection, making it ideal for many IoT applications.

From here, you could explore several exciting enhancements:

*   **Use a Custom BLE Service**: For more complex scenarios, you could move from advertising data to a full GATT service with custom characteristics for accelerometer and gyroscope data. This would require a BLE connection but allows for two-way communication and more structured data access.
*   **Implement Sensor Fusion**: The raw accelerometer and gyroscope data are useful, but their real power comes from combining them. You could implement a sensor fusion algorithm (like Madgwick or Mahony) on the device to calculate its absolute orientation (roll, pitch, yaw) and broadcast that instead.
*   **Power Management**: Zephyr offers extensive power management features. You could put the device into a deep sleep state between sensor reads and broadcasts to dramatically extend battery life.
*   **Data Visualization**: On the receiving end, you could build a simple web or desktop application to visualize the incoming data in real-time, creating a virtual 3D model that mimics the physical device's orientation.

Happy hacking!
