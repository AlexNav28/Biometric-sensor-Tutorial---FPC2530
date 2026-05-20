# Getting Started with the FPC2534 Fingerprint Sensor over I2C (Qwiic)

**Alexis Navarrete — ECE 196 SP26**

---

## Tutorial Brainstorm

This tutorial walks through setting up and using the **SparkFun Fingerprint Sensor – FPC2534 (Qwiic)** over I2C with an ESP32-S3 microcontroller. The FPC2534 is part of the FPC AllKey Pro Biometric System family by Fingerprint Cards AB — a complete system-in-package that bundles a capacitive fingerprint sensor, an MCU, flash storage, and onboard authentication software into a tiny 11 mm round LGA package. The SparkFun breakout board (SEN-29854) exposes the sensor's I2C interface through a Qwiic connector, making it plug-and-play for prototyping. The goal is to demonstrate the three core capabilities of the sensor — **fingerprint enrollment**, **fingerprint identification/authentication**, and **touch navigation** — each with its own standalone code example, and then combine everything into a single unified demo at the end. One important caveat: at the time of writing, the SparkFun FPC2534 Arduino library is **not available** through the Arduino IDE Library Manager or PlatformIO's library registry, so manual library installation is required (downloading the ZIP from GitHub and placing it in your libraries folder). The I2C implementation is also only supported on **ESP32** and **RP2 (RP2040/RP2350)** platforms because the FPC2534's I2C transaction structure requires low-level bus access that the standard Arduino Wire library doesn't support — SparkFun bundles custom I2C helper functions for those two platforms only. +++ Needs more details about the UART and SPI ++++

---

## Table of Contents

1. [Objectives](#objectives)
2. [Background — About the FPC2534 Sensor](#background--about-the-fpc2534-sensor)
3. [Supplies](#supplies)
4. [Hardware Setup](#hardware-setup)
5. [Library Installation (Manual)](#library-installation-manual)
6. [Firmware — Initialization over I2C](#firmware--initialization-over-i2c)
7. [Feature 1 — Fingerprint Enrollment](#feature-1--fingerprint-enrollment)
8. [Feature 2 — Fingerprint Identification](#feature-2--fingerprint-identification)
9. [Feature 3 — Touch Navigation](#feature-3--touch-navigation)
10. [Combined Demo — All Features Together](#combined-demo--all-features-together)
11. [Troubleshooting](#troubleshooting)
12. [Conclusion](#conclusion)
13. [References](#references)

---

## Objectives

By the end of this tutorial you will be able to:

- Understand the hardware architecture of the FPC2534 AllKey Pro biometric sensor
- Wire the SparkFun Qwiic breakout to an ESP32-S3 over I2C
- Manually install the SparkFun FPC2534 Arduino library
- Initialize the sensor and handle its **callback-driven** messaging pattern
- Enroll, identify, and use navigation gestures through individual code examples
- Combine all features into one cohesive application

---

## Background — About the FPC2534 Sensor

The SparkFun breakout board is built around the **FPC2534AP** — the LGA variant of the AllKey Pro family from Fingerprint Cards AB. Here are some key hardware specs pulled straight from the [FPC2530 product specification (SPC27317/5)](https://www.fingerprints.com):

| Parameter | Value |
|---|---|
| Sensor diameter | 11.0 mm |
| Active sensing area | 5.0 × 4.8 mm (100 × 96 pixels) |
| Spatial resolution | 508 dpi |
| Supply voltage (VDD) | 3.0 – 3.3 – 3.6 V |
| Deep sleep current | 14 µA |
| Finger detect current | 22 µA |
| Active current | 5.2 mA (typ) |
| I²C speed | Up to 400 kbit/s (Fast mode) |
| Max enrolled fingerprints | 30 |
| Enrollment touches | 12 – 16 |
| Identify time | 140 – 1400 ms |
| FAR | 1 / 100,000 per finger |
| FRR (after template update) | < 1.5 % |
| IP rating | IP67 |
| Operating temperature | −40 °C to +125 °C |

The FPC2534 is the *high-security* variant of the family, supporting **SPI, UART, USB, and I²C** interfaces. Interface selection is controlled by the `IF_CFG_1` and `IF_CFG_2` pins — for I²C, both must be pulled **HIGH**. The SparkFun board handles this via onboard jumpers.

> **Important:** The FPC2534 runs on **3.3 V only**. Supplying 5 V directly (e.g., from USB VBUS) will damage the sensor.

### Block Diagram

The sensor integrates a capacitive fingerprint sensor, ADC, MCU, flash memory, GPIOs, a hidden ESD bezel, and an interface multiplexer — all inside a single molded SiP package. Internal communication lines between the sensor, flash, and MCU are hidden inside the package body and cannot be probed externally.

![FPC2534 Block Diagram](https://raw.githubusercontent.com/sparkfun/SparkFun_Qwiic_Fingerprint_Sensor_FPC2534/main/docs/assets/img/Fingerprint_Sensor_Qwiic-GHBanner.png)
*SparkFun Fingerprint Sensor – FPC2534 (Qwiic) breakout board. Image credit: SparkFun Electronics.*

---

## Supplies

You will need the following components:

- **1×** [SparkFun Fingerprint Sensor – FPC2534 (Qwiic)](https://www.sparkfun.com/sparkfun-fingerprint-sensor-fpc2534-qwiic.html) — SEN-29854
- **1×** ESP32-S3 development board (e.g., SparkFun Thing Plus ESP32-S3, or any ESP32/RP2040/RP2350 board)
- **1×** Qwiic cable (or 4-pin JST-SH cable)
- **2×** Jumper wires for `RST` and `IRQ` connections
- **1×** USB-C cable for programming
- A computer with **Arduino IDE 2.x** installed

---

## Hardware Setup

### Wiring Diagram

Connect the SparkFun FPC2534 breakout to your ESP32-S3 as follows:

| FPC2534 Breakout Pin | ESP32-S3 Pin | Purpose |
|---|---|---|
| **SDA** (Qwiic) | GPIO 8 (or board SDA) | I²C Data |
| **SCL** (Qwiic) | GPIO 9 (or board SCL) | I²C Clock |
| **3V3** (Qwiic) | 3.3 V | Power |
| **GND** (Qwiic) | GND | Ground |
| **RST** | GPIO 4 | Hardware reset (active low) |
| **IRQ** | GPIO 5 | Interrupt request — sensor signals data ready |

> **Note on I²C pull-ups:** The Qwiic connector already includes pull-up resistors on the breakout board. No external pull-ups are needed.

Make sure the **interface selection jumpers** on the SparkFun board are set to **I²C mode** (both CFG1 and CFG2 = HIGH). Refer to the [SparkFun hookup guide](https://docs.sparkfun.com/SparkFun_Qwiic_Fingerprint_Sensor_FPC2534/) for jumper locations.

---

## Library Installation (Manual)

> **At the time of Creatig this Tutorial, the SparkFun FPC2534 Arduino library is NOT available through the Arduino IDE Library Manager or PlatformIO's library registry.** You must install it manually.

### Steps

1. Go to the GitHub repository: [https://github.com/sparkfun/SparkFun_FPC2534_Arduino_Library](https://github.com/sparkfun/SparkFun_FPC2534_Arduino_Library)
2. Click **Code → Download ZIP**
3. In Arduino IDE, go to **Sketch → Include Library → Add .ZIP Library…**
4. Select the downloaded ZIP file
5. Restart Arduino IDE

Alternatively, extract the ZIP contents into your Arduino `libraries/` folder:

```
~/Arduino/libraries/SparkFun_FPC2534_Arduino_Library/
├── examples/
├── src/
├── library.properties
└── ...
```

### Platform Support

The I²C interface for the FPC2534 is **only supported on ESP32 and RP2 (RP2040/RP2350) boards**. The FPC2534's I²C transactions involve a dynamic payload structure that the standard Arduino `Wire` library cannot handle. SparkFun provides custom low-level I²C helper functions for these two platforms.

---

## Firmware — Initialization over I2C

The FPC2534 uses a **callback-driven messaging pattern** — you don't call a function and get a return value like typical sensors. Instead, you register callback functions that the library invokes when events happen (finger detected, enrollment progress, match result, etc.). In every iteration of `loop()`, you call `processNextResponse()` to check for and dispatch incoming messages.

### Basic I2C Initialization

```cpp
#include <SparkFun_FPC2534.h>

// --- Pin Definitions ---
#define IRQ_PIN     5
#define RST_PIN     4
#define I2C_BUS     0   // I2C bus number (typically 0)

// --- Sensor Object ---
SfeFPC2534I2C mySensor;

// --- Callback structure ---
sfDevFPC2534Callbacks_t myCallbacks;

// --- Callbacks ---
void onReady(bool isReady) {
    if (isReady)
        Serial.println("Sensor is ready!");
}

void onError(uint16_t errorCode) {
    Serial.print("Sensor error: 0x");
    Serial.println(errorCode, HEX);
}

void setup() {
    Serial.begin(115200);
    delay(1000);

    // Set up RST pin
    pinMode(RST_PIN, OUTPUT);
    digitalWrite(RST_PIN, HIGH);

    // Initialize I2C
    Wire.begin();

    // Register callbacks
    myCallbacks.on_is_ready_change = onReady;
    myCallbacks.on_error = onError;

    // Initialize the sensor
    bool status = mySensor.begin(SFE_FPC2534_I2C_ADDRESS, Wire, I2C_BUS, IRQ_PIN);

    if (!status) {
        Serial.println("Failed to initialize FPC2534!");
        while (1);
    }

    // Register callbacks with the library
    mySensor.setCallbacks(myCallbacks);

    // Reset the sensor to ensure clean state
    mySensor.reset();

    Serial.println("Waiting for sensor to be ready...");
}

void loop() {
    // Process incoming messages from the sensor
    mySensor.processNextResponse();
}
```

> **Ping warning:** If you "ping" the I²C bus at the FPC2534's address (a common Arduino pattern using `Wire.beginTransmission()` / `Wire.endTransmission()`), the sensor enters an unknown state. Always call `mySensor.reset()` after any ping operation.

---

## Feature 1 — Fingerprint Enrollment

Enrollment stores a new fingerprint template on the sensor's internal flash. The process requires **12–16 finger touches**, with the user slightly shifting their finger position between each touch to capture more of the fingerprint surface.

```cpp
// --- Enrollment Callback ---
void onEnroll(uint8_t samplesRemaining, uint8_t feedback) {
    if (samplesRemaining == 0) {
        Serial.println("\nEnrollment complete!");
    } else {
        Serial.print("Samples remaining: ");
        Serial.println(samplesRemaining);
        Serial.println("Lift and re-place your finger...");
    }
}

// --- Status Callback (tracks finger lift during enrollment) ---
void onStatus(uint16_t event, uint16_t state) {
    if (event == EVENT_FINGER_LOST && mySensor.currentMode() == STATE_ENROLL) {
        Serial.print(".");  // Progress indicator
    }
}

// In setup(), add:
//   myCallbacks.on_enroll = onEnroll;
//   myCallbacks.on_status = onStatus;

// To start enrollment (call after sensor is ready):
void startEnrollment() {
    Serial.println("Starting enrollment — place your finger on the sensor.");
    mySensor.requestEnroll();  // Optional: pass a specific template ID
}
```

The sensor can store **up to 30 fingerprint templates**. If you need to clear them, use `requestDeleteTemplate()` or `requestDeleteAllTemplates()`.

---

## Feature 2 — Fingerprint Identification

After enrolling one or more fingerprints, the sensor can identify/verify a finger press against stored templates.

```cpp
// --- Identify Callback ---
void onIdentify(bool matched, uint16_t templateId) {
    if (matched) {
        Serial.print("Match! Template ID: ");
        Serial.println(templateId);
    } else {
        Serial.println("No match.");
    }
}

// In setup(), add:
//   myCallbacks.on_identify = onIdentify;

// To start identification:
void startIdentify() {
    Serial.println("Place your finger to identify...");
    mySensor.requestIdentify();
}
```

**Timing:** Identification takes **140 ms** (best case — matching against a specific ID) to **1400 ms** (worst case — 30 enrolled templates, match on the last one).

**Lockout:** After 5 consecutive failed attempts (configurable), the sensor locks out for 15 seconds by default.

---

## Feature 3 — Touch Navigation

The FPC2534 can also function as a miniature touchpad, detecting swipe gestures (up/down/left/right), short presses, and long presses.

```cpp
// --- Navigation Callback ---
void onNavigation(uint8_t gesture) {
    switch (gesture) {
        case NAV_SWIPE_UP:    Serial.println("Swipe UP");    break;
        case NAV_SWIPE_DOWN:  Serial.println("Swipe DOWN");  break;
        case NAV_SWIPE_LEFT:  Serial.println("Swipe LEFT");  break;
        case NAV_SWIPE_RIGHT: Serial.println("Swipe RIGHT"); break;
        case NAV_PRESS:       Serial.println("Press");        break;
        case NAV_LONG_PRESS:  Serial.println("Long Press");   break;
        default:              Serial.println("Unknown");      break;
    }
}

// In setup(), add:
//   myCallbacks.on_navigation = onNavigation;

// To start navigation mode:
void startNavigation() {
    Serial.println("Entering navigation mode — swipe or press the sensor.");
    mySensor.startNavigationMode();
}
```

Navigation gestures are detected based on how the finger **leaves** the sensor surface. For example, if the top edge loses contact first, the sensor interprets it as a *swipe down*. A straight lift is a *press*, and holding the finger for about 1 second triggers a *long press*.

---

## Combined Demo — All Features Together

This final example integrates enrollment, identification, and navigation into a single interactive serial menu.

```cpp
#include <SparkFun_FPC2534.h>

#define IRQ_PIN     5
#define RST_PIN     4
#define I2C_BUS     0

SfeFPC2534I2C mySensor;
sfDevFPC2534Callbacks_t myCallbacks;
bool sensorReady = false;

// ---- Callbacks ----
void onReady(bool ready) {
    sensorReady = ready;
    if (ready) {
        Serial.println("\n=== FPC2534 Ready ===");
        printMenu();
    }
}

void onError(uint16_t err) {
    Serial.print("[ERROR] 0x"); Serial.println(err, HEX);
}

void onEnroll(uint8_t remaining, uint8_t feedback) {
    if (remaining == 0) {
        Serial.println("\nEnrollment complete!");
        printMenu();
    } else {
        Serial.print("  Touches remaining: "); Serial.println(remaining);
    }
}

void onIdentify(bool matched, uint16_t id) {
    if (matched) {
        Serial.print("Matched template ID: "); Serial.println(id);
    } else {
        Serial.println("No match found.");
    }
    printMenu();
}

void onNavigation(uint8_t gesture) {
    const char* names[] = {"UNKNOWN", "UP", "DOWN", "LEFT", "RIGHT", "PRESS", "LONG_PRESS"};
    uint8_t idx = (gesture <= 6) ? gesture : 0;
    Serial.print("  Navigation: "); Serial.println(names[idx]);
}

void onStatus(uint16_t event, uint16_t state) {
    if (event == EVENT_FINGER_LOST) Serial.print(".");
}

// ---- Menu ----
void printMenu() {
    Serial.println("\n--- Menu ---");
    Serial.println("  [e] Enroll a finger");
    Serial.println("  [i] Identify a finger");
    Serial.println("  [n] Navigation mode (press 'q' to stop)");
    Serial.println("  [l] List templates");
    Serial.println("  [d] Delete all templates");
    Serial.print("> ");
}

void setup() {
    Serial.begin(115200);
    delay(1000);

    pinMode(RST_PIN, OUTPUT);
    digitalWrite(RST_PIN, HIGH);
    Wire.begin();

    myCallbacks.on_is_ready_change = onReady;
    myCallbacks.on_error           = onError;
    myCallbacks.on_enroll          = onEnroll;
    myCallbacks.on_identify        = onIdentify;
    myCallbacks.on_navigation      = onNavigation;
    myCallbacks.on_status          = onStatus;

    if (!mySensor.begin(SFE_FPC2534_I2C_ADDRESS, Wire, I2C_BUS, IRQ_PIN)) {
        Serial.println("FPC2534 init failed!"); while (1);
    }

    mySensor.setCallbacks(myCallbacks);
    mySensor.reset();
    Serial.println("Initializing FPC2534...");
}

void loop() {
    mySensor.processNextResponse();

    if (Serial.available() && sensorReady) {
        char cmd = Serial.read();
        switch (cmd) {
            case 'e':
                Serial.println("Starting enrollment — place finger on sensor.");
                mySensor.requestEnroll();
                break;
            case 'i':
                Serial.println("Place finger to identify...");
                mySensor.requestIdentify();
                break;
            case 'n':
                Serial.println("Navigation mode active. Press 'q' + Enter to exit.");
                mySensor.startNavigationMode();
                break;
            case 'q':
                Serial.println("Stopping navigation mode.");
                mySensor.requestAbort();
                printMenu();
                break;
            case 'l':
                Serial.println("Requesting template list...");
                mySensor.requestListTemplates();
                break;
            case 'd':
                Serial.println("Deleting all templates...");
                mySensor.requestDeleteAllTemplates();
                break;
        }
    }
}
```

---

## Troubleshooting

- **Sensor doesn't respond:** Double-check jumper settings for I²C mode. Make sure both `CFG1` and `CFG2` are set to HIGH on the breakout board.
- **Library not found:** The library is *not* in the Arduino Library Manager. You must install it manually from the [GitHub repo](https://github.com/sparkfun/SparkFun_FPC2534_Arduino_Library).
- **I²C doesn't work on your board:** Only **ESP32** and **RP2 (RP2040/RP2350)** are supported for I²C. If you're on a different platform, use UART or SPI instead.
- **Sensor enters unknown state after ping:** Always call `mySensor.reset()` after any I²C bus scan or ping operation.
- **Enrollment fails repeatedly:** Make sure the finger fully covers the sensor and is placed flat and parallel. Shift the finger slightly between touches to capture a larger area.

---

## Conclusion

The FPC2534 is a seriously capable sensor packed into a tiny form factor — enrollment, identification with a 1:100,000 false accept rate, encrypted template storage, navigation gestures, IP67 protection, and 14 µA deep sleep, all in an 11 mm round package. The SparkFun Qwiic breakout makes wiring dead simple, though the library situation requires a bit of manual work for now. Once you get past the setup, the callback-driven API is intuitive and maps cleanly onto event-driven embedded applications like smart locks, crypto wallets, and access control systems.

---

## References

- [FPC2530 Product Specification (SPC27317/5)](https://www.fingerprints.com) — Fingerprint Cards AB
- [SparkFun Fingerprint Sensor – FPC2534 (Qwiic) — Product Page](https://www.sparkfun.com/sparkfun-fingerprint-sensor-fpc2534-qwiic.html)
- [SparkFun FPC2534 Board GitHub Repo](https://github.com/sparkfun/SparkFun_Qwiic_Fingerprint_Sensor_FPC2534)
- [SparkFun FPC2534 Arduino Library](https://github.com/sparkfun/SparkFun_FPC2534_Arduino_Library)
- [SparkFun Hookup Guide](https://docs.sparkfun.com/SparkFun_Qwiic_Fingerprint_Sensor_FPC2534/)
