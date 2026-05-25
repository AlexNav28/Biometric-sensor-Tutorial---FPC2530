# Getting Started with the FPC2534 Fingerprint Sensor over I2C (Qwiic)

**Alexis Navarrete — ECE 196 SP26**

---

## Abstract

This tutorial covers how to get the **SparkFun Fingerprint Sensor – FPC2534 (Qwiic)** up and running over I2C on an ESP32-S3. The FPC2534 packs a capacitive fingerprint sensor, an MCU, flash, and full authentication logic into a single 11 mm round chip. We'll go through the sensor's three main features — **enrollment**, **identification**, and **touch navigation** — with individual code examples for each, then combine everything into one interactive demo. The theory section ties in capacitive sensing principles from ECE 35 (Circuits and Systems) to explain what's actually happening inside the sensor when it reads a fingerprint.

---

## Background — About the FPC2534 Sensor

The SparkFun breakout is built around the **FPC2534AP** — the LGA variant of the AllKey Pro family from Fingerprint Cards AB. Here are the key specs from the [FPC2530 datasheet](https://cdn.sparkfun.com/assets/b/7/4/f/6/SPC27317-5_FPC2530.pdf):

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

It supports **SPI, UART, USB, and I²C**. Which interface is active depends on the `IF_CFG_1` and `IF_CFG_2` pins — for I²C, both need to be HIGH. The SparkFun board has jumpers for this so you don't have to wire it yourself.

> **Important:** The FPC2534 runs on **3.3 V only**. Supplying 5 V directly (e.g., from USB VBUS) will damage the sensor.

### Schematic

The SparkFun breakout board routes the FPC2534's I²C, UART, SPI, and USB interfaces through connectors, includes a 3.3 V LDO regulator (RT9080), I²C pull-ups (2.2 kΩ), interface selection jumpers (CFG1/CFG2), and ESD bezel grounding. The full KiCad schematic is shown below:

The full schematic is available as a PDF: [SparkFun Qwiic Fingerprint Sensor FPC2534 Schematic (PDF)](https://cdn.sparkfun.com/assets/0/5/d/5/9/SparkFun_Qwiic_Fingerprint_Sensor_FPC2534.pdf). Source: [SparkFun GitHub](https://github.com/sparkfun/SparkFun_Qwiic_Fingerprint_Sensor_FPC2534).

### The Sensor on a Board

![FPC2534 on SparkFun Breakout](https://www.sparkfun.com/media/.renditions/wysiwyg/Documentation/29854-Action-ProductPage.jpg)
*The FPC2534 sensor mounted on the SparkFun Qwiic breakout board, connected via Qwiic cable. Image credit: SparkFun Electronics.*

---

## Intro Concept / Theory — Capacitive Sensing

So how does the FPC2534 actually "see" a fingerprint? It comes down to **capacitive sensing** — and the physics behind it maps directly to stuff from **ECE 35 (Circuits and Systems)**.

### How It Works

Under the sensor surface there's a grid of tiny capacitor plates — one per pixel. When you put your finger down, the ridges of your fingerprint sit right on the surface (small gap), and the valleys hover above it (bigger gap). That difference in distance changes the capacitance at each pixel.

From ECE 35, the parallel-plate capacitance equation:

$$C = \varepsilon \cdot \frac{A}{d}$$

**C** is capacitance, **ε** is permittivity, **A** is plate area, **d** is the gap. At a ridge, the skin is basically touching the surface (small **d**, higher **C**). At a valley, there's an air gap (larger **d**, lower **C**). The sensor's ADC reads the charge on each pixel and you get a grayscale image — ridges show up bright, valleys dark.

![Capacitive Fingerprint Sensing Principle](https://www.mdpi.com/sensors/sensors-18-02200/article_deploy/html/images/sensors-18-02200-g001.png)
*Capacitive fingerprint sensor cross-section showing how ridges and valleys create different capacitance values at each pixel. Image source: Maltoni et al., "Fingerprint Recognition," Sensors, 2018. ([MDPI](https://www.mdpi.com/1424-8220/18/7/2200))*

### Charge Transfer

The FPC2534 measures these tiny capacitance differences using **charge transfer** — another concept from ECE 35's RC circuit analysis. A known reference capacitor gets charged to a set voltage, then gets connected to the unknown pixel capacitor. Charge flows between them (conservation of charge, Q = CV), and the resulting voltage tells you the pixel's capacitance. The sensor repeats this multiple times per pixel to average out noise — same idea as averaging N measurements to cut noise by √N.

### Why This Matters

The FPC2534's array is 100 × 96 pixels at 508 dpi, so each pixel is roughly 50 µm × 50 µm. At that scale, you're looking at femtofarad-level capacitance differences between ridges and valleys. Resolving that reliably is why the FPC2534 has built-in shielding, an ESD bezel, and runs at 3.3 V — all to keep electrical noise from drowning out the signal.

---

## Supplies

You will need the following components:

- **1×** [SparkFun Fingerprint Sensor – FPC2534 (Qwiic)](https://www.sparkfun.com/sparkfun-fingerprint-sensor-fpc2534-qwiic.html) — SEN-29854
- **1×** ESP32-S3 development board (e.g., Adafruit Feather ESP32-S3, or any ESP32/RP2040/RP2350 board with a Qwiic connector)
- **1×** Qwiic cable (or 4-pin JST-SH cable)
- **2×** Jumper wires for `RST` and `IRQ` connections
- **1×** USB-C cable for programming
- A computer with **PlatformIO** installed (via VS Code extension)

---

## Hardware Setup

### Wiring Diagram

Connect the SparkFun FPC2534 breakout to your ESP32-S3 as follows. This example uses the **Adafruit Feather ESP32-S3**, where the Qwiic connector uses SDA = GPIO 3 and SCL = GPIO 4 by default.

| FPC2534 Breakout Pin | ESP32-S3 Pin | Purpose |
|---|---|---|
| **SDA** (Qwiic) | GPIO 3 (Feather default) | I²C Data |
| **SCL** (Qwiic) | GPIO 4 (Feather default) | I²C Clock |
| **3V3** (Qwiic) | 3.3 V | Power |
| **GND** (Qwiic) | GND | Ground |
| **RST** | GPIO 5 | Hardware reset (active low) |
| **IRQ** | GPIO 6 | Interrupt request — sensor signals data ready |

> **Important:** Do not use GPIO 3 or GPIO 4 for RST/IRQ — those are already taken by I²C through the Qwiic connector.

> **Note on I²C pull-ups:** The Qwiic connector already includes pull-up resistors on the breakout board. No external pull-ups are needed.

Make sure the **interface selection jumpers** on the SparkFun board are set to **I²C mode** (both CFG1 and CFG2 = HIGH). Refer to the [SparkFun hookup guide](https://docs.sparkfun.com/SparkFun_Qwiic_Fingerprint_Sensor_FPC2534/) for jumper locations.

---

## PlatformIO Project Setup

### What is PlatformIO?

[PlatformIO](https://platformio.org/) is an open-source embedded development platform that runs as an extension inside **VS Code**. Unlike the Arduino IDE, PlatformIO gives you full control over your project structure — you choose exactly which libraries and platform versions your project compiles against through a single `platformio.ini` config file, so builds are reproducible and nothing breaks when a library updates. It also integrates natively with **Git and GitHub** (since your project is just a normal folder with source files), supports **IntelliSense** for code autocompletion, and handles board definitions, toolchains, and serial monitoring all in one place.

### Why PlatformIO over Arduino IDE?

For this tutorial and the ECE 196 final project, PlatformIO was chosen because it offers direct control over build flags (critical for getting USB serial working on the Feather ESP32-S3), a proper project directory structure (`src/`, `lib/`, `include/`) that plays well with version control, and the ability to pin specific library versions so your code doesn't break when dependencies update. The Arduino IDE works fine for quick sketches, but for a multi-file embedded project with specific hardware requirements, PlatformIO's flexibility and VS Code integration make development significantly smoother.

### Installation

1. Install [Visual Studio Code](https://code.visualstudio.com/)
2. Open VS Code, go to the **Extensions** tab (Ctrl+Shift+X)
3. Search for **"PlatformIO IDE"** and click **Install**
4. After installation, click the PlatformIO icon in the sidebar → **New Project**
5. Select your board (e.g., "Adafruit Feather ESP32-S3"), framework "Arduino", and create

### Project Configuration

Configure your `platformio.ini` as follows:

```ini
[env:adafruit_feather_esp32s3]
platform = espressif32
board = adafruit_feather_esp32s3
framework = arduino
monitor_speed = 115200
build_flags = 
    -DARDUINO_USB_CDC_ON_BOOT=1
    -DARDUINO_USB_MODE=1
```

> **Why these build flags?** The Adafruit Feather ESP32-S3 uses native USB (not a separate USB-to-serial chip). Without `-DARDUINO_USB_CDC_ON_BOOT=1`, `Serial` output goes nowhere and `while (!Serial)` hangs forever.

### Installing the Library

Install the SparkFun FPC2534 library through the PlatformIO Library Manager: search for **"SparkFun FPC2534"** in the PIO Home Libraries tab. Alternatively, add it to your `platformio.ini`:

```ini
lib_deps =
    sparkfun/SparkFun FPC2534 Arduino Library
```

### Platform Support

The I²C interface for the FPC2534 is **only supported on ESP32 and RP2 (RP2040/RP2350) boards**. The FPC2534's I²C transactions involve a dynamic payload structure that the standard Arduino `Wire` library cannot handle. SparkFun provides custom low-level I²C helper functions for these two platforms.

---

## Firmware — Initialization over I2C

The FPC2534 doesn't work like most sensors where you call a function and get a value back. Instead, it uses a **callback-driven pattern** — you register functions that get called when stuff happens (finger detected, enrollment done, match result, etc.). Every loop iteration, you call `processNextResponse()` to check for new messages from the sensor and dispatch them to your callbacks.

### Setup Sequence

Getting the sensor initialized is a bit particular — the order matters and there's a quirk you need to work around. This sequence comes straight from the [SparkFun Example02_EnrollI2C](https://github.com/sparkfun/SparkFun_FPC2534_Arduino_Library/blob/main/examples/Example02_EnrollI2C/Example02_EnrollI2C.ino):

1. `Wire.begin()` — fire up the I²C bus
2. Hardware reset — toggle RST low then high
3. I²C ping — check the sensor is actually on the bus
4. `mySensor.begin()` — init the library
5. Register your callbacks via `setCallbacks()`
6. Hardware reset *again* — because the ping in step 3 puts the sensor in a weird state

### Basic I2C Initialization

```cpp
#include <Arduino.h>
#include <Wire.h>
#include "SparkFun_FPC2534.h"

// --- Pin Definitions (Adafruit Feather ESP32-S3) ---
#define IRQ_PIN 6
#define RST_PIN 5
#define I2C_BUS 0

// --- Sensor Object ---
SfeFPC2534I2C mySensor;

// --- Callback structure ---
static sfDevFPC2534Callbacks_t cmd_cb = {0};

// --- Hardware Reset (toggle RST pin) ---
void reset_sensor(void) {
    mySensor.clearData();
    pinMode(RST_PIN, OUTPUT);
    digitalWrite(RST_PIN, LOW);
    delay(10);
    digitalWrite(RST_PIN, HIGH);
    delay(250);
}

// --- Callbacks ---
static void on_is_ready_change(bool isReady) {
    if (isReady)
        Serial.println("[STARTUP]\tFPC2534 Device is ready");
}

static void on_error(uint16_t error) {
    Serial.print("[ERROR]\tSensor Error Code: ");
    Serial.println(error);
    reset_sensor();
}

void setup() {
    delay(2000);
    Serial.begin(115200);
    while (!Serial) { ; }

    // 1. Init I2C
    Wire.begin();

    // 2. First reset
    reset_sensor();

    // 3. Ping check
    Wire.beginTransmission(kFPC2534DefaultAddress);
    if (Wire.endTransmission() != 0) {
        Serial.println("[ERROR]\tFPC2534 not found on I2C bus. HALT");
        while (1) delay(1000);
    }
    Serial.println("[STARTUP]\tFPC2534 found on I2C bus");

    // 4. Initialize the library
    if (!mySensor.begin(kFPC2534DefaultAddress, Wire, I2C_BUS, IRQ_PIN)) {
        Serial.println("[ERROR]\tFPC2534 init failed. HALT.");
        while (1) delay(1000);
    }
    Serial.println("[STARTUP]\tFPC2534 initialized.");

    // 5. Register callbacks AFTER begin()
    cmd_cb.on_is_ready_change = on_is_ready_change;
    cmd_cb.on_error = on_error;
    mySensor.setCallbacks(cmd_cb);

    // 6. Second reset — workaround after ping
    reset_sensor();

    Serial.println("[STARTUP]\tWaiting for sensor ready callback...");
}

void loop() {
    fpc_result_t rc = mySensor.processNextResponse();
    if (rc != FPC_RESULT_OK && rc != FPC_PENDING_OPERATION) {
        Serial.print("[ERROR] Processing Error: ");
        Serial.println(rc);
        reset_sensor();
    }
    delay(200);
}
```

> **Ping warning:** If you "ping" the I²C bus at the FPC2534's address (using `Wire.beginTransmission()` / `Wire.endTransmission()`), the sensor enters an unknown state. Always do a hardware reset (toggle RST pin LOW → HIGH) after any ping operation.

---

## Feature 1 — Fingerprint Enrollment

Enrollment saves a new fingerprint template to the sensor's internal flash. It takes **12–16 touches** — you place your finger, lift it, shift it slightly, and place it again. This lets the sensor build up a more complete picture of your fingerprint.

To kick off enrollment, call `requestEnroll()` with an `fpc_id_type_t` struct. The library hits your `on_enroll` callback after each touch with how many samples are left.

```cpp
// --- Enrollment Callback ---
// NOTE: Parameter order is (feedback, samples_remaining) per the library API
static void on_enroll(uint8_t feedback, uint8_t samples_remaining) {
    if (samples_remaining == 0) {
        Serial.println("..done!");
        Serial.println("[INFO]\tRemove finger from sensor.");
        delay(500);
        // Sync template count from sensor
        mySensor.requestListTemplates();
    } else {
        Serial.print(samples_remaining);
        Serial.print(".");
    }
}

// --- Status Callback (tracks finger lift during enrollment) ---
static void on_status(uint16_t event, uint16_t state) {
    if (mySensor.currentMode() == STATE_ENROLL && event == EVENT_FINGER_LOST) {
        Serial.print(".");  // Progress indicator
    }
}

// In setup(), register both:
//   cmd_cb.on_enroll = on_enroll;
//   cmd_cb.on_status = on_status;

// To start enrollment (call after sensor is ready):
void startEnrollment() {
    Serial.println("Starting enrollment — place your finger on the sensor.");
    mySensor.setLED(true);
    fpc_id_type_t id = {0};
    id.type = ID_TYPE_GENERATE_NEW;  // Auto-assign template ID
    id.id = 0;
    fpc_result_t rc = mySensor.requestEnroll(id);
    if (rc != FPC_RESULT_OK) {
        Serial.print("[ERROR]\tFailed to start enroll: ");
        Serial.println(rc);
    }
}
```

The sensor can store **up to 30 fingerprint templates**. If you need to clear them, use `requestDeleteTemplate()` with `ID_TYPE_ALL`:

```cpp
fpc_id_type_t id = {0};
id.type = ID_TYPE_ALL;
mySensor.requestDeleteTemplate(id);
```

---

## Feature 2 — Fingerprint Identification

Once you've enrolled at least one finger, you can run identification — the sensor checks a finger press against all stored templates and tells you if there's a match. Call `requestIdentify()` with `ID_TYPE_ALL` to compare against everything enrolled.

```cpp
// --- Identify Callback ---
static void on_identify(bool is_match, uint16_t id) {
    Serial.print(is_match ? "" : " NO ");
    Serial.print(" MATCH ");

    if (is_match) {
        Serial.print("  {Template ID: ");
        Serial.print(id);
        Serial.print("}");
    }
    Serial.println();
    Serial.println("[INFO]\tRemove finger from sensor.");
}

// In setup(), register:
//   cmd_cb.on_identify = on_identify;

// To start identification:
void startIdentify() {
    Serial.println("Place your finger to identify...");
    fpc_id_type_t id = {0};
    id.type = ID_TYPE_ALL;  // Compare against all enrolled templates
    id.id = 0;
    fpc_result_t rc = mySensor.requestIdentify(id, 1);  // Second arg is an operation tag
    if (rc != FPC_RESULT_OK) {
        Serial.print("[ERROR]\tFailed to start identify: ");
        Serial.println(rc);
    }
}
```

**Timing:** Identification takes **140 ms** (best case — matching against a specific ID) to **1400 ms** (worst case — 30 enrolled templates, match on the last one).

**I²C hang note:** When using I²C, the sensor can hang during identification until the finger is removed. If `on_status()` fires with `EVENT_IMAGE_READY` while `currentMode() == STATE_IDENTIFY`, prompt the user to lift their finger.

---

## Feature 3 — Touch Navigation

Besides fingerprint stuff, the FPC2534 doubles as a tiny touchpad. It can detect swipes (up/down/left/right), taps, and long presses. You start it with `startNavigationMode()` and pass an orientation value (0–3, each step is 90°).

```cpp
// --- Navigation Callback ---
// NOTE: gesture type is uint16_t, and constants use the CMD_NAV_EVENT_* prefix
static void on_navigation(uint16_t gesture) {
    Serial.print("[NAV]\t");
    switch (gesture) {
        case CMD_NAV_EVENT_UP:         Serial.println("UP");         break;
        case CMD_NAV_EVENT_DOWN:       Serial.println("DOWN");       break;
        case CMD_NAV_EVENT_LEFT:       Serial.println("LEFT");       break;
        case CMD_NAV_EVENT_RIGHT:      Serial.println("RIGHT");      break;
        case CMD_NAV_EVENT_PRESS:      Serial.println("PRESS");      break;
        case CMD_NAV_EVENT_LONG_PRESS: Serial.println("LONG PRESS"); break;
        default:                       Serial.println("UNKNOWN");    break;
    }
}

// In setup(), register:
//   cmd_cb.on_navigation = on_navigation;

// To start navigation mode:
void startNavigation() {
    Serial.println("Navigation mode — swipe or tap the sensor.");
    fpc_result_t rc = mySensor.startNavigationMode(0);  // 0 = default orientation
    if (rc != FPC_RESULT_OK) {
        Serial.print("[ERROR]\tFailed to start nav: ");
        Serial.println(rc);
    }
}

// To stop navigation mode:
void stopNavigation() {
    mySensor.requestAbort();
}
```

The gestures work based on how your finger **leaves** the sensor. If the top edge lifts off first, the sensor reads that as a swipe down. A straight lift-off is a tap, and keeping your finger down for about a second triggers a long press.

---

## Combined Demo — All Features Together

This puts enrollment, identification, and navigation together into one sketch with a serial menu. The code follows the same patterns from the official SparkFun [Example01 (Navigation)](https://github.com/sparkfun/SparkFun_FPC2534_Arduino_Library/blob/main/examples/Example01_NavigationI2C/Example01_NavigationI2C.ino) and [Example02 (Enroll)](https://github.com/sparkfun/SparkFun_FPC2534_Arduino_Library/blob/main/examples/Example02_EnrollI2C/Example02_EnrollI2C.ino), just merged into a single interactive program.

```cpp
#include <Arduino.h>
#include <Wire.h>
#include "SparkFun_FPC2534.h"

//----------------------------------------------------------------------------
// Pin Config — update these to match YOUR wiring
//----------------------------------------------------------------------------
#define IRQ_PIN 6
#define RST_PIN 5
#define I2C_BUS 0

//----------------------------------------------------------------------------
// Global state
//----------------------------------------------------------------------------
SfeFPC2534I2C mySensor;

uint16_t numberOfTemplates = 0;
bool isInitialized = false;
bool drawTheMenu = false;
bool ledState = false;
unsigned long lastMenuDrawTime = 0;

//----------------------------------------------------------------------------
// reset_sensor()
//----------------------------------------------------------------------------
void reset_sensor(void) {
    mySensor.clearData();
    pinMode(RST_PIN, OUTPUT);
    digitalWrite(RST_PIN, LOW);
    delay(10);
    digitalWrite(RST_PIN, HIGH);
    delay(250);
}

//----------------------------------------------------------------------------
// drawMenu()
//----------------------------------------------------------------------------
static void drawMenu() {
    drawTheMenu = false;
    lastMenuDrawTime = millis();
    mySensor.setLED(false);

    Serial.println();
    Serial.println("----------------------------------------------------------------");
    Serial.println(" FPC2534 Combined Demo");
    Serial.println("----------------------------------------------------------------");
    Serial.println();
    Serial.print(" Enrolled templates: ");
    Serial.print(numberOfTemplates);
    Serial.println(" / 30");
    Serial.println();
    Serial.println(" Select an option:");
    Serial.println("\t1)  Enroll a new fingerprint");
    Serial.println("\t2)  Erase all fingerprint templates");
    Serial.println("\t3)  Validate a fingerprint");
    Serial.println("\t4)  Start navigation mode");
    Serial.println("\t5)  List templates");
    Serial.println();
    Serial.print("> ");
}

//------------------------------------------------------------------------------------
// Callbacks
//------------------------------------------------------------------------------------
static void on_error(uint16_t error) {
    // Error 70 = FPC_RESULT_PROTOCOL_VERSION_ERROR
    // Known I2C artifact — stale bus data after identify. Silent reset.
    if (error == FPC_RESULT_PROTOCOL_VERSION_ERROR) {
        reset_sensor();
        return;
    }
    Serial.print("[ERROR]\tSensor Error Code: ");
    Serial.println(error);
    reset_sensor();
}

static void on_is_ready_change(bool isReady) {
    if (isReady) {
        Serial.println("[STARTUP]\tFPC2534 Device is ready");
        fpc_result_t rc = mySensor.requestListTemplates();
        if (rc != FPC_RESULT_OK) {
            Serial.print("[ERROR]\tFailed to get template list: ");
            Serial.println(rc);
        }
    }
}

static void on_enroll(uint8_t feedback, uint8_t samples_remaining) {
    if (samples_remaining == 0) {
        Serial.println("..done!");
        Serial.println("[INFO]\tRemove finger from sensor.");
        delay(500);
        mySensor.requestListTemplates();
    } else {
        Serial.print(samples_remaining);
        Serial.print(".");
    }
}

static void on_identify(bool is_match, uint16_t id) {
    Serial.print(is_match ? "" : " NO ");
    Serial.print(" MATCH ");
    if (is_match) {
        Serial.print("  {Template ID: ");
        Serial.print(id);
        Serial.print("}");
    }
    Serial.println();
    Serial.println("[INFO]\tRemove finger from sensor.");
    drawTheMenu = true;
}

static void on_list_templates(uint16_t num_templates, uint16_t *template_ids) {
    numberOfTemplates = num_templates;
    if (num_templates > 0 && template_ids != nullptr) {
        Serial.print("[INFO]\tStored IDs: ");
        for (uint16_t i = 0; i < num_templates; i++) {
            Serial.print(template_ids[i]);
            if (i < num_templates - 1) Serial.print(", ");
        }
        Serial.println();
    }
    isInitialized = true;
    drawTheMenu = true;
}

static void on_navigation(uint16_t gesture) {
    Serial.print("[NAV]\t");
    switch (gesture) {
        case CMD_NAV_EVENT_NONE:       Serial.println("NONE");       break;
        case CMD_NAV_EVENT_UP:         Serial.println("UP");         break;
        case CMD_NAV_EVENT_DOWN:       Serial.println("DOWN");       break;
        case CMD_NAV_EVENT_RIGHT:      Serial.println("RIGHT");      break;
        case CMD_NAV_EVENT_LEFT:       Serial.println("LEFT");       break;
        case CMD_NAV_EVENT_PRESS:
            Serial.print("PRESS -> {LED ");
            Serial.print(ledState ? "OFF" : "ON");
            Serial.println("}");
            ledState = !ledState;
            mySensor.setLED(ledState);
            break;
        case CMD_NAV_EVENT_LONG_PRESS:
            Serial.println("LONG PRESS -> returning to menu");
            mySensor.requestAbort();
            drawTheMenu = true;
            break;
        default: Serial.println("UNKNOWN"); break;
    }
}

static void on_version(char *version) {
    Serial.print("[INFO]\tFirmware: ");
    Serial.println(version);
}

static void on_status(uint16_t event, uint16_t state) {
    if (mySensor.currentMode() == 0) {
        if (event == EVENT_FINGER_LOST)
            drawTheMenu = true;
        else if ((state & STATE_APP_FW_READY) == STATE_APP_FW_READY) {
            if (event == EVENT_NONE)
                drawTheMenu = true;
            else if (event == EVENT_IDLE && isInitialized)
                drawTheMenu = true;
        }
    }
    else if (mySensor.currentMode() == STATE_IDENTIFY && event == EVENT_IMAGE_READY) {
        Serial.println(" -\tRemove finger and try again");
    }
    else if (mySensor.currentMode() == STATE_ENROLL && event == EVENT_FINGER_LOST) {
        Serial.print(".");
    }
}

//----------------------------------------------------------------------------
// Callback structure
//----------------------------------------------------------------------------
static sfDevFPC2534Callbacks_t cmd_cb = {0};

//------------------------------------------------------------------------------------
// setup()
//------------------------------------------------------------------------------------
void setup() {
    delay(2000);
    Serial.begin(115200);
    while (!Serial) { ; }

    Serial.println();
    Serial.println("----------------------------------------------------------------");
    Serial.println(" FPC2534 Combined Demo");
    Serial.println("----------------------------------------------------------------");
    Serial.println();

    Wire.begin();
    reset_sensor();

    Wire.beginTransmission(kFPC2534DefaultAddress);
    if (Wire.endTransmission() != 0) {
        Serial.println("[ERROR]\tFPC2534 not found on I2C bus. HALT");
        while (1) delay(1000);
    }
    Serial.println("[STARTUP]\tFPC2534 found on I2C bus");

    if (!mySensor.begin(kFPC2534DefaultAddress, Wire, I2C_BUS, IRQ_PIN)) {
        Serial.println("[ERROR]\tFPC2534 init failed. HALT.");
        while (1) delay(1000);
    }
    Serial.println("[STARTUP]\tFPC2534 initialized.");

    cmd_cb.on_error = on_error;
    cmd_cb.on_status = on_status;
    cmd_cb.on_enroll = on_enroll;
    cmd_cb.on_identify = on_identify;
    cmd_cb.on_navigation = on_navigation;
    cmd_cb.on_version = on_version;
    cmd_cb.on_list_templates = on_list_templates;
    cmd_cb.on_is_ready_change = on_is_ready_change;
    mySensor.setCallbacks(cmd_cb);

    reset_sensor();
    Serial.println("[STARTUP]\tSystem ready. Waiting for sensor callback...");
}

//------------------------------------------------------------------------------------
// loop()
//------------------------------------------------------------------------------------
void loop() {
    fpc_result_t rc = mySensor.processNextResponse();
    if (rc != FPC_RESULT_OK && rc != FPC_PENDING_OPERATION) {
        Serial.print("[ERROR] Processing Error: ");
        Serial.println(rc);
        reset_sensor();
    }
    else if (drawTheMenu && (millis() - lastMenuDrawTime > 1000)) {
        drawMenu();
    }

    if (Serial.available()) {
        char cmd = Serial.read();
        if (cmd == '\n' || cmd == '\r') {
            // skip
        }
        else if (cmd == '1') {
            Serial.println("\n Starting enrollment — place and remove finger repeatedly.");
            mySensor.setLED(true);
            fpc_id_type_t id = {0};
            id.type = ID_TYPE_GENERATE_NEW;
            id.id = 0;
            fpc_result_t r = mySensor.requestEnroll(id);
            if (r != FPC_RESULT_OK) {
                Serial.print("[ERROR]\tFailed to start enroll: ");
                Serial.println(r);
                drawTheMenu = true;
            } else
                Serial.print("\t samples remaining 12..");
        }
        else if (cmd == '2') {
            if (numberOfTemplates == 0) {
                Serial.println("[INFO]\tNo templates to delete");
                drawTheMenu = true;
            } else {
                Serial.println("\n Deleting all templates...");
                fpc_id_type_t id = {0};
                id.type = ID_TYPE_ALL;
                id.id = 0;
                fpc_result_t r = mySensor.requestDeleteTemplate(id);
                if (r != FPC_RESULT_OK) {
                    Serial.print("[ERROR]\tFailed to delete: ");
                    Serial.println(r);
                } else
                    numberOfTemplates = 0;
            }
        }
        else if (cmd == '3') {
            if (numberOfTemplates == 0) {
                Serial.println("[INFO]\tNo templates to validate against");
                drawTheMenu = true;
            } else {
                Serial.println("\n Place finger for validation...");
                fpc_id_type_t id = {0};
                id.type = ID_TYPE_ALL;
                id.id = 0;
                fpc_result_t r = mySensor.requestIdentify(id, 1);
                if (r != FPC_RESULT_OK) {
                    Serial.print("[ERROR]\tFailed to start identify: ");
                    Serial.println(r);
                }
            }
        }
        else if (cmd == '4') {
            Serial.println("\n Navigation mode — swipe or tap. Long press to exit.");
            fpc_result_t r = mySensor.startNavigationMode(0);
            if (r != FPC_RESULT_OK) {
                Serial.print("[ERROR]\tFailed to start nav: ");
                Serial.println(r);
                drawTheMenu = true;
            }
        }
        else if (cmd == '5') {
            Serial.println("\n Requesting template list...");
            mySensor.requestListTemplates();
        }
    }

    delay(200);
}
```

---

## Troubleshooting

- **Sensor doesn't respond:** Double-check jumper settings for I²C mode. Make sure both `CFG1` and `CFG2` are set to HIGH on the breakout board.
- **No serial output (Feather ESP32-S3):** Add `-DARDUINO_USB_CDC_ON_BOOT=1` and `-DARDUINO_USB_MODE=1` to your `build_flags` in `platformio.ini`. The Feather uses native USB — without these flags, `Serial` goes nowhere.
- **Library not found:** Search for "SparkFun FPC2534" in the PlatformIO Library Manager, or add `sparkfun/SparkFun FPC2534 Arduino Library` to `lib_deps` in `platformio.ini`.
- **I²C doesn't work on your board:** Only **ESP32** and **RP2 (RP2040/RP2350)** are supported for I²C. If you're on a different platform, use UART or SPI instead.
- **Sensor enters unknown state after ping:** Always do a hardware reset (toggle RST pin LOW → HIGH) after any I²C bus scan or ping operation.
- **Enrollment fails repeatedly:** Make sure the finger fully covers the sensor and is placed flat and parallel. Shift the finger slightly between touches to capture a larger area.
- **Sensor hangs during identify (I²C only):** The sensor may hang until the finger is removed. If `on_status()` fires with `EVENT_IMAGE_READY` while in identify mode, prompt the user to remove their finger.
- **Error code 70 after identify:** This is `FPC_RESULT_PROTOCOL_VERSION_ERROR` — stale I²C bus data, not a real problem. The combined demo silently resets on this error.
- **RST/IRQ pin conflicts with I²C:** On the Adafruit Feather ESP32-S3, the Qwiic connector uses GPIO 3 (SDA) and GPIO 4 (SCL). Do not assign RST or IRQ to these pins.

---

## Example — Connection to Final Project

This tutorial ties directly into my ECE 196 final project: the **Intelligent Wallet**. It's a biometric smart wallet running on an ESP32-S3-Mini-1 that uses the FPC2534 as its main way to authenticate the owner. When a fingerprint match succeeds, a servo motor unlatches the wallet — if it doesn't match, the wallet stays locked. Enrollment happens through a React Native companion app over BLE, and all the fingerprint matching runs on-device through the FPC2534's built-in template storage.

The navigation feature covered in this tutorial isn't currently part of the wallet, but it's something I'd like to add in a future version — using swipe gestures to scroll through wallet status info on a small display, or a long press to trigger the "find my wallet" buzzer.
---

## Additional Resources

- [FPC2530 Product Specification (SPC27317/5)](https://cdn.sparkfun.com/assets/b/7/4/f/6/SPC27317-5_FPC2530.pdf) — Fingerprint Cards AB
- [SparkFun Fingerprint Sensor – FPC2534 (Qwiic) — Product Page](https://www.sparkfun.com/sparkfun-fingerprint-sensor-fpc2534-qwiic.html)
- [SparkFun FPC2534 Board GitHub Repo](https://github.com/sparkfun/SparkFun_Qwiic_Fingerprint_Sensor_FPC2534)
- [SparkFun FPC2534 Arduino Library](https://github.com/sparkfun/SparkFun_FPC2534_Arduino_Library)
- [SparkFun Hookup Guide](https://docs.sparkfun.com/SparkFun_Qwiic_Fingerprint_Sensor_FPC2534/)
- [Capacitive Sensing Explained — Texas Instruments Application Note](https://www.ti.com/lit/an/snoa927/snoa927.pdf)
- [PlatformIO Documentation](https://docs.platformio.org/)

**Course tie-in:** The capacitive sensing and charge transfer concepts in this tutorial come from **ECE 35 — Circuits and Systems** at UC San Diego (specifically HW 06 on capacitors/inductors and HW 07a on first-order RC circuits).

---

## AI-Use Disclosure

I used Claude (by Anthropic) to help proofread and clean up the writing. I also used it to cross-check all the code examples against the actual SparkFun FPC2534 Arduino Library source on GitHub — making sure callback signatures, method parameters, constant names, and the initialization sequence matched the real API. The hardware wiring, theory write-up, and project integration were done by me using the SparkFun hookup guide and the FPC2534 datasheet as references.