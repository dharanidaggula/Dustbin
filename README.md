# 🗑️ Smart Dustbin Monitoring System

A Streamlit dashboard that monitors the fill level of multiple smart dustbins.
The prototype uses **simulated sensor data**, and is structured so an
**ESP8266 + ultrasonic sensor** can send real readings later.

## Run it

```bash
pip install -r requirements.txt
streamlit run app.py
```

Open http://localhost:8501

## Features

- Sample bins: BIN-001, BIN-002, BIN-003 (edit `SAMPLE_BINS` in `app.py`)
- Cards showing Bin ID, location, fill %, status and last-updated time
- Status: 0-79% Normal, 80-89% Nearly Full, 90-100% Full
- Warning banner at 80%+, full-bin alert at 90%+
- Sidebar: pick a bin, **Simulate Waste Added**, set level manually, empty bin
- Line chart of fill-level history for the selected bin
- Optional email alerts (disabled by default)

## Optional: email alerts

In `app.py`, edit `EMAIL_CONFIG`, set `"enabled": True`, and use a Gmail
*app password* (never your real password, never commit it). Then tick the
checkbox in the sidebar. One email is sent each time a bin enters
Nearly Full or Full.

## Upgrade path: real hardware

```
Ultrasonic Sensor -> ESP8266 -> REST API -> app.py -> Streamlit
```

The REST API is already built into `app.py` (standard library only) but is off by default.

```bash
# Windows PowerShell:  $env:ENABLE_API="1"; streamlit run app.py
# Mac/Linux:
ENABLE_API=1 streamlit run app.py
```

Test it without hardware:

```bash
curl -X POST http://localhost:8502/api/reading \
  -H "Content-Type: application/json" \
  -d '{"bin_id": "BIN-001", "fill_level": 72}'
```

Or send the raw ultrasonic distance and let the server convert it
(calibrate `SENSOR_CONFIG` to your bin's depth):

```bash
curl -X POST http://localhost:8502/api/reading \
  -H "Content-Type: application/json" \
  -d '{"bin_id": "BIN-002", "distance_cm": 12.5}'
```

`GET http://localhost:8502/api/bins` returns all bins as JSON.

### ESP8266 sketch (Arduino IDE)

```cpp
#include <ESP8266WiFi.h>
#include <ESP8266HTTPClient.h>
#include <WiFiClient.h>

const char* ssid = "YOUR_WIFI";
const char* pass = "YOUR_PASSWORD";
const char* url  = "http://YOUR_LAPTOP_IP:8502/api/reading";
const int trigPin = D1, echoPin = D2;

void setup() {
  pinMode(trigPin, OUTPUT); pinMode(echoPin, INPUT);
  WiFi.begin(ssid, pass);
  while (WiFi.status() != WL_CONNECTED) delay(500);
}

float readDistanceCm() {
  digitalWrite(trigPin, LOW);  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH); delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  return pulseIn(echoPin, HIGH) * 0.0343 / 2;
}

void loop() {
  WiFiClient client; HTTPClient http;
  http.begin(client, url);
  http.addHeader("Content-Type", "application/json");
  String body = "{\"bin_id\":\"BIN-001\",\"distance_cm\":" + String(readDistanceCm()) + "}";
  http.POST(body);
  http.end();
  delay(5000);
}
```

Laptop and ESP8266 must be on the same Wi-Fi, and your firewall must allow port 8502.

## Hackathon tip

Get the simulation demo working perfectly first. If hardware misbehaves on
presentation day, you still have a complete working dashboard.
