# Greenhouse-
## What This Project Does

| S.No | Title | How it's covered |
|------|-------|-----------------|
| 1 | Temp & Humidity Monitoring | DHT11 → GPIO4, alerts sent via WebSocket |
| 6 | Smart Ventilation Control | Fan on Relay1, Servo vent flap on GPIO13 |
| 9 | IoT Dashboard | Real-time WebSocket dashboard (HTML app) |
| 10 | Automated Climate Control | Feedback loop: sensor → threshold → actuator |
| 11 | Soil Moisture Prediction | ADC read, sparkline history, auto-pump |

---

## Hardware Required

| Component | Qty | Pin |
|-----------|-----|-----|
| ESP32 38-pin | 1 | — |
| DHT11 | 1 | GPIO4 |
| Soil Moisture Sensor (resistive) | 1 | GPIO34 (ADC) |
| PIR Sensor | 1 | GPIO14 |
| 10K Potentiometer with knob | 1 | GPIO35 (ADC) |
| 5V Channel Relay Module (x2 on one board) | 1 | GPIO26, GPIO27 |
| Servo SG90 | 1 | GPIO13 |
| LED + 220Ω resistor | 1 | GPIO2 |
| LM2596 Buck Module (12V→5V) | 1 | Powers relay + servo |
| Breadboard, jumper wires | — | — |

---

## Wiring Summary

### Power
- Use the LM2596 module to step down your 12V supply to 5V
- 5V rail → Relay module VCC, Servo VCC (red wire)
- ESP32 powered via USB (5V) separately or from the 5V rail via Vin pin
- All GNDs tied together (common ground)

### Sensors (3.3V logic — safe for ESP32)
```
DHT11:
  VCC  → 3.3V
  DATA → GPIO4 (add 10K pull-up resistor between DATA and 3.3V)
  GND  → GND

Soil Moisture Sensor:
  VCC  → 3.3V
  AO   → GPIO34
  GND  → GND

PIR Sensor:
  VCC  → 3.3V (or 5V depending on module; check your module)
  OUT  → GPIO14
  GND  → GND

Potentiometer:
  Left pin  → GND
  Right pin → 3.3V
  Middle (wiper) → GPIO35
```

### Actuators (5V powered, controlled by ESP32 GPIO)
```
Relay Module (5V CH optocoupler relay board):
  VCC → 5V (from LM2596)
  GND → GND
  IN1 → GPIO26  (Fan control — Active LOW)
  IN2 → GPIO27  (Pump control — Active LOW)

Servo SG90:
  Red  → 5V (from LM2596)
  Brown → GND
  Orange (signal) → GPIO13

Status LED:
  Anode → 220Ω → GPIO2
  Cathode → GND
```

---

## Software Setup

### Arduino IDE
1. Install ESP32 board support:
   - File → Preferences → Additional Boards Manager URLs:
     `https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`
   - Tools → Board Manager → search "esp32" → install

2. Install libraries (Sketch → Include Library → Manage Libraries):
   - **DHT sensor library** by Adafruit
   - **Adafruit Unified Sensor**
   - **WebSockets** by Markus Sattler (arduinoWebSockets)
   - **ESP32Servo**
   - **ArduinoJson** by Benoit Blanchon

3. Open `greenhouse_esp32.ino`

4. Edit these lines to match your Wi-Fi:
   ```cpp
   #define WIFI_SSID     "YOUR_WIFI_SSID"
   #define WIFI_PASSWORD "YOUR_WIFI_PASSWORD"
   ```
   Or uncomment `#define USE_AP_MODE` — the ESP32 will create a hotspot
   named `Greenhouse_ESP32` (password: `greenhouse123`)

5. Select Board: **ESP32 Dev Module** (or your specific 38-pin variant)
   - Port: select the COM port for your ESP32
   - Upload speed: 115200

6. Upload. Open Serial Monitor at 115200 baud.
   - Note the IP address printed (e.g., `Connected! IP: 192.168.1.100`)

---

## Dashboard Setup

1. Open `greenhouse_dashboard.html` in any browser (Chrome recommended)
2. Enter the ESP32's IP address in the "ESP32 IP" field
3. Click **Connect**
4. The dashboard will connect and start showing live data

**If using AP Mode:**
- Connect your phone/laptop to Wi-Fi network `Greenhouse_ESP32`
- Enter IP `192.168.4.1` in the dashboard
- Click Connect

**Demo Mode:**
- Click **Demo Mode** to simulate sensor data without hardware

---

## How Automation Works

```
Every 2 seconds:
  1. Read DHT11 → temperature + humidity
  2. Read GPIO34 ADC → soil moisture
  3. Read GPIO14 → PIR motion
  4. Read GPIO35 ADC → pot (manual/auto mode select)

  If AUTO mode (pot below midpoint):
    Temperature > 32°C  → Fan ON  + Vent OPEN (servo 90°)
    Humidity > 85%      → Fan ON  + alert
    Soil dry (ADC>2500) → Pump ON for 10 seconds
    PIR triggered       → Alert flag + LED blinks

  If MANUAL mode (pot above midpoint):
    Pot position → directly sets vent angle (0–180°)
    Dashboard toggles control fan/pump

  5. Broadcast JSON over WebSocket to all connected clients
  6. Dashboard updates in real-time
```

---

## Communication Protocol

```
ESP32 IP:Port 80  → HTTP REST API
  GET /data         → returns current JSON state
  GET /setThreshold?tempHigh=32&humidHigh=85&...
  GET /manualFan?state=1
  GET /manualPump?state=1
  GET /manualVent?state=1

ESP32 IP:Port 81  → WebSocket (bi-directional)
  ESP32 → browser: JSON state every 2s
  Browser → ESP32: JSON command
    {"fan": true}
    {"pump": false}
    {"vent": true}
    {"tempHigh": 30, "humidHigh": 80}
```

---

## Project Objectives Met

- **Minimal human intervention**: fully automated feedback loop
- **Remote monitoring**: any device on same Wi-Fi opens dashboard
- **Real-time alerts**: WebSocket pushes alerts instantly  
- **Manual override**: physical potentiometer for lab demos
- **Scalable**: can add more sensors (CO2, light) by adding to JSON payload
