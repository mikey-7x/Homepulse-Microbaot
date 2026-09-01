# Homepulse Microbaot 🏠⚡

**Homepulse** is a professional-grade, local IoT home automation system powered by a Wemos D1 Mini (ESP8266) and a custom Android dashboard. It features real-time state synchronization, Active-LOW hardware logic handling, and a dynamic, vector-animated Android UI that reacts instantly to physical hardware changes.

## ✨ Features
* **Live Animated Dashboard:** Android UI features glowing neon vector graphics for active relays and real-time spinning animations for fans.
* **3-Channel PWM Fan Control:** Independent power toggles and 0-100% speed sliders for up to 3 fans.
* **6-Channel Relay Management:** Active-LOW logic control for sockets, bulbs, and tubelights.
* **Real-Time Sync:** JSON-based state polling ensures the app perfectly matches the physical hardware state.
* **Secure Network Config:** Wemos IP settings in the app are locked behind an admin password (`1234qwer`).
* **Fully Offline:** Runs entirely on the local Wemos hotspot (`Wemos_Home_Auto`) with no cloud dependency.

## 🛠️ Hardware Pinout
The system is designed for the **Wemos D1 Mini** combined with an external Relay board and PWM motor drivers. All logic is configured for **Active-LOW** hardware.

| Device Type | Wemos Pin | UI Designation |
| :--- | :--- | :--- |
| **Relay 1** | `D0` | Socket 1 |
| **Relay 2** | `D3` | Tubelight 1 |
| **Relay 3** | `D4` | Socket 2 |
| **Relay 4** | `D6` | Bulb 1 |
| **Relay 5** | `D7` | Tubelight 2 |
| **Relay 6** | `D8` | Socket 3 |
| **Fan 1 (PWM)** | `D1` | Fan 1 |
| **Fan 2 (PWM)** | `D5` | Fan 2 |
| **Fan 3 (PWM)** | `D2` | Fan 3 |

*(Note: `D8` must not be pulled HIGH during boot. Ensure your relay board does not interfere with the ESP8266 boot sequence).*

## 📱 Android App Installation
The Homepulse Android app features a dark-themed glass UI with dynamic SVG animations. 

1. Go to the **[Releases](../../releases)** section of this repository.
2. Download `Homepulse.apk`.
3. Install the APK on your Android device.
4. Connect to the Wemos Wi-Fi Hotspot (SSID: `Wemos_Home_Auto`, Password: `87651234`).
5. Open the app to begin controlling your hardware.

## ⚙️ Wemos D1 Mini Firmware (C++)
Upload the following code to your Wemos D1 Mini using the Arduino IDE.

```cpp
#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>

const char* ssid = "Wemos_Home_Auto";
const char* password = "87651234"; 

ESP8266WebServer server(80);

// Pins & States for 3 Fans
const int fanPins[3] = {D1, D5, D2}; 
int fanSpeeds[3] = {0, 0, 0}; 
bool fanStates[3] = {false, false, false}; 

// Pins & States for 6 Relays
const int relayPins[6] = {D0, D3, D4, D6, D7, D8};
bool relayStates[6] = {false, false, false, false, false, false}; 

void setup() {
  Serial.begin(115200);

  for (int i = 0; i < 3; i++) {
    pinMode(fanPins[i], OUTPUT);
    analogWrite(fanPins[i], 1023); // 1023 = Active-LOW OFF
  }

  for (int i = 0; i < 6; i++) {
    pinMode(relayPins[i], OUTPUT);
    digitalWrite(relayPins[i], HIGH); // HIGH = Active-LOW OFF
  }

  WiFi.softAP(ssid, password);
  Serial.print("Hotspot Started. AP IP: ");
  Serial.println(WiFi.softAPIP());

  server.on("/relay", HTTP_GET, handleRelay);
  server.on("/fan_state", HTTP_GET, handleFanState);
  server.on("/fan_speed", HTTP_GET, handleFanSpeed);
  server.on("/status", HTTP_GET, handleStatus);
  server.begin();
}

void loop() {
  server.handleClient();
}

void updateFanHardware(int id) {
  if (!fanStates[id]) {
    analogWrite(fanPins[id], 1023); 
  } else {
    int pwmValue = map(fanSpeeds[id], 0, 100, 1023, 0); 
    analogWrite(fanPins[id], pwmValue);
  }
}

void handleRelay() {
  if (server.hasArg("id") && server.hasArg("state")) {
    int id = server.arg("id").toInt(); 
    String state = server.arg("state");

    if (id >= 0 && id < 6) {
      if (state == "on") {
        digitalWrite(relayPins[id], LOW); 
        relayStates[id] = true;
      } else {
        digitalWrite(relayPins[id], HIGH); 
        relayStates[id] = false;
      }
      server.send(200, "text/plain", "OK");
      return;
    }
  }
  server.send(400, "text/plain", "Bad Request");
}

void handleFanState() {
  if (server.hasArg("id") && server.hasArg("state")) {
    int id = server.arg("id").toInt(); 
    String state = server.arg("state");

    if (id >= 0 && id < 3) {
      fanStates[id] = (state == "on");
      updateFanHardware(id);
      server.send(200, "text/plain", "OK");
      return;
    }
  }
  server.send(400, "text/plain", "Bad Request");
}

void handleFanSpeed() {
  if (server.hasArg("id") && server.hasArg("speed")) {
    int id = server.arg("id").toInt(); 
    int speed = server.arg("speed").toInt(); 

    if (id >= 0 && id < 3 && speed >= 0 && speed <= 100) {
      fanSpeeds[id] = speed;
      updateFanHardware(id);
      server.send(200, "text/plain", "OK");
      return;
    }
  }
  server.send(400, "text/plain", "Bad Request");
}

void handleStatus() {
  String json = "{";
  json += "\"r\":[";
  for(int i = 0; i < 6; i++) {
    json += (relayStates[i] ? "true" : "false");
    if(i < 5) json += ",";
  }
  json += "],\"fs\":[";
  for(int i = 0; i < 3; i++) {
    json += (fanStates[i] ? "true" : "false");
    if(i < 2) json += ",";
  }
  json += "],\"f\":[";
  json += fanSpeeds[0]; json += ","; json += fanSpeeds[1]; json += ","; json += fanSpeeds[2];
  json += "]}";
  
  server.send(200, "application/json", json);
}
```
---
🏗️ Built With
 * Watari-ARM64-Studio - The Android application for this project was developed, scaffolded, and compiled entirely on an Android mobile device using the Watari ARM64 Gradle/Kotlin Forge.
 * Arduino C++ - Wemos D1 Mini backend logic.
---
## 👨‍🔬 Author Info

**Project Creator:** *Mikey-7x / Yogesh R. Chauhan*  
**GitHub:** [github.com/mikey-7x](https://github.com/mikey-7x)  
**License:** [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)  
**Date:** 1 September 2026

Special thanks to the open-source community for providing awesome libraries!
