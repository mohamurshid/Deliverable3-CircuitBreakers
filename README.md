# ICS 4111: Embedded Systems & IoT — Semester Project, Deliverable 3

**Title:** Cloud-Based Environmental & Gas Monitoring System using ESP32, DHT22 and MQ-5
**Course:** ICS 4111 (Apr–Jul 2026)
**Date:** *[submission date]*
**Class section:** *[ICS 4.1A / 4.1B / 4.1C]*

**Members:**
| Reg No. | Name |
|---|---|
| 169004 | Mohammed Ilhan Hamud |
| 169962 | Abdalla Muhammad Murshid |
| 154210 | Elvis Wafuke |
| 159540 | Wanjohi Joy Wambui |
| 150851 | Ndeti Melissa Mwende |
| 156740 | Gitonga Benson Kamau |

---

## 1. Objective

This deliverable extends the Deliverable 2 prototype by adding **cloud connectivity**. The ESP32 reads temperature, humidity, and gas concentration data from a DHT22 and a gas sensor (MQ-5 in concept; simulated using Wokwi's MQ2-type gas sensor part), displays live readings on an OLED screen, and transmits each reading over Wi-Fi to **InfluxDB Cloud**, a time-series database. The stored data is then visualised on a **Grafana Cloud** dashboard containing three or more panels.

---

## 2. System Architecture

```
 ┌────────────┐        ┌──────────────────────┐        ┌────────────────┐
 │   DHT22    │──data──▶│                      │        │                │
 ├────────────┤        │                      │  Wi-Fi │  InfluxDB Cloud│
 │ Gas sensor │──AO────▶│        ESP32         │───HTTPS▶│ (time-series   │
 ├────────────┤        │  (reads + displays    │  POST  │   storage)     │
 │ SSD1306    │◀──I2C──│   + sends to cloud)   │        └───────┬────────┘
 │   OLED     │        │                      │                │ query
 └────────────┘        └──────────────────────┘                ▼
                                                          ┌────────────────┐
                                                          │  Grafana Cloud │
                                                          │  (dashboard,   │
                                                          │  3+ panels)    │
                                                          └────────────────┘
```

**Components:**
| Component | Function | ESP32 Pin |
|---|---|---|
| DHT22 | Temperature & humidity sensing | GPIO 15 (data, with 10 kΩ pull-up to 3V3) |
| Gas sensor (MQ-5 / Wokwi MQ2-type) | Combustible/LPG gas concentration (analog) | GPIO 34 (ADC1_CH6) |
| SSD1306 OLED (128×64, I2C) | Local readout of live values | GPIO 21 (SDA), GPIO 22 (SCL) |
| ESP32 DevKit (board-esp32-devkit-c-v4) | Microcontroller, Wi-Fi, cloud client | — |

> **Note on wiring:** our original hand-drawn schematic (submitted for Deliverable 2) routed the DHT22 data line and OLED I2C lines incorrectly and referenced a gas sensor part that doesn't exist in Wokwi's component library. For Deliverable 3 we rebuilt the circuit using verified Wokwi parts and pin names — see `Circuit 3.png` and the Wokwi project link below.

---

## 3. Hardware / Simulation Setup

This project was simulated on **Wokwi** (online ESP32 simulator), as permitted by the assignment brief.

- **Wokwi project (public link):** https://wokwi.com/projects/468276398579020801
- Components used in the simulation: `board-esp32-devkit-c-v4`, `wokwi-dht22`, `wokwi-gas-sensor`, `board-ssd1306`, three resistors (10 kΩ pull-up for DHT22, 1 kΩ series + 1 kΩ pull-down for the gas sensor's analog line).
- Libraries (via Wokwi Library Manager / `libraries.txt`): `DHT sensor library for ESPx` (DHTesp — required for reliable DHT22 readings on the simulated ESP32), `Adafruit GFX Library`, `Adafruit SSD1306`, `ESP8266 Influxdb` (InfluxDB Arduino client by Tobias Schürg — also works on ESP32).

### 3.1 Wiring diagram (corrected)

![Wiring diagram](Circuit%203.png)

| From | To |
|---|---|
| ESP32 `3V3` | DHT22 `VCC`, OLED `VCC` |
| ESP32 `GND` | DHT22 `GND`, gas sensor `GND`, OLED `GND` |
| ESP32 `GPIO15` | DHT22 `DATA` (+10 kΩ pull-up to 3V3) |
| ESP32 `5V` | Gas sensor `VCC` |
| ESP32 `GPIO34` | Gas sensor `AO` (via series/pull-down resistors) |
| ESP32 `GPIO22` | OLED `SCL` |
| ESP32 `GPIO21` | OLED `SDA` |

---

## 4. Firmware (ESP32 Code)

Full source code is included in this repository at [`sketch.ino`](./sketch.ino). Key logic:

1. Connect to Wi-Fi (`Wokwi-GUEST` network when run in simulation).
2. Read DHT22 temperature & humidity (via the `DHTesp` library), and the gas sensor's analog value, on each loop.
3. Display all readings + Wi-Fi status on the OLED.
4. Every 10 seconds, package the latest readings as an InfluxDB **Point** (measurement `environment_data`, tags `device` and `location`, fields `temperature`, `humidity`, `gas_raw`, `gas_voltage`) and write it to InfluxDB Cloud over HTTPS using `InfluxDbClient.h`.

```cpp
// excerpt — full file in sketch.ino
sensorPoint.clearFields();
sensorPoint.addField("temperature", temperature);
sensorPoint.addField("humidity", humidity);
sensorPoint.addField("gas_raw", gasRaw);
sensorPoint.addField("gas_voltage", gasVoltage);

if (!client.writePoint(sensorPoint)) {
  Serial.print("InfluxDB write failed: ");
  Serial.println(client.getLastErrorMessage());
}
```

**Credentials** (Wi-Fi SSID/password, InfluxDB URL/token/org/bucket) are stored as `#define` constants at the top of the sketch. The version of `sketch.ino` in this repository uses placeholder values for the token/org — these were filled in with our actual InfluxDB Cloud credentials only inside the live Wokwi project (link above), not committed here.

---

## 5. Cloud Storage — InfluxDB Cloud

1. Created a free **InfluxDB Cloud** account at `https://cloud2.influxdata.com`.
2. Created a bucket named `iot_sensors` to hold all sensor data.
3. Generated an **All-Access API Token** under *Load Data → API Tokens*.
4. Copied the **InfluxDB URL**, **Organization ID**, and **Bucket name** from *Load Data → Client Libraries → Arduino* into the firmware.
5. Verified incoming data using the InfluxDB **Data Explorer / SQL query view**, querying the `environment_data` measurement.

**Evidence:**

![InfluxDB data verification](Influx%20DB.png)

Sample query used to verify data (InfluxDB 3 SQL view):

```sql
SELECT *
FROM "environment_data"
WHERE
  time >= now() - interval '1 hour'
  AND ("gas_raw" IS NOT NULL OR "gas_voltage" IS NOT NULL OR "humidity" IS NOT NULL)
```

Equivalent Flux query (used in Grafana, see Section 6):

```flux
from(bucket: "iot_sensors")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "environment_data")
```

---

## 6. Cloud Visualisation — Grafana Cloud

1. Created a free **Grafana Cloud** account at `https://grafana.com`.
2. Added a new **InfluxDB data source** (*Connections → Data sources → Add new connection → InfluxDB*), using:
   - Query language: **Flux**
   - URL: our InfluxDB Cloud region URL
   - Organization: our InfluxDB org
   - Default bucket: `iot_sensors`
   - Token: our InfluxDB API token (read access)
3. Verified the connection with **Save & Test**.
4. Built a dashboard named **"ICS4111 IoT Environmental Monitor"** with the following panels:

| Panel | Type | Purpose |
|---|---|---|
| Temperature over time | Time series | Trend of DHT22 temperature readings |
| Humidity over time | Time series | Trend of DHT22 humidity readings |
| Gas concentration (raw ADC) | Gauge | Current level of gas sensor reading |
| Current readings | Stat panel | Latest temperature, humidity, gas value at a glance |

**Evidence:**

![Grafana dashboard](Grafana%20Dashboard.png)

---

## 7. Results

- DHT22 and gas sensor readings successfully reported into InfluxDB Cloud at 10-second intervals (e.g. 31.3 °C, 78.5% humidity confirmed in testing).
- The OLED correctly displayed live values and Wi-Fi connectivity status.
- The Grafana dashboard correctly visualised temperature, humidity, and gas readings.
- The InfluxDB SQL/Data Explorer view confirmed rows landing in the `environment_data` measurement with all expected fields (device, gas_raw, gas_voltage, humidity, location, temperature).
- Gas sensor readings remained near zero throughout testing since Wokwi's simulated gas sensor defaults to a clean-air concentration; this is expected behaviour in simulation, not a fault.

---

## 8. Repository Structure

```
.
├── README.md            <- this file
├── sketch.ino            <- ESP32 source code
├── diagram.json           <- Wokwi wiring diagram
├── libraries.txt          <- required Arduino libraries
├── Circuit 3.png          <- wiring/circuit screenshot
├── Influx DB.png          <- InfluxDB data verification screenshot
├── Grafana Dashboard.png  <- Grafana dashboard screenshot
└── groupwork-photo.jpg    <- evidence of groupwork
```

> ⚠️ **Security note:** real Wi-Fi credentials and InfluxDB tokens are not committed to this repository — `sketch.ino` here uses placeholder values; the live Wokwi project (linked above) holds the working credentials.

---

## 9. Evidence of Groupwork

![Groupwork photo](groupwork-photo.jpeg)

All group members listed above contributed to this deliverable. Contribution breakdown:

| Member | Contribution |
|---|---|
| Mohammed Ilhan Hamud | Circuit design & Wokwi simulation |
| Abdalla Muhammad Murshid | ESP32 firmware (sensor reading + OLED) |
| Elvis Wafuke | InfluxDB Cloud integration |
| Wanjohi Joy Wambui | Grafana dashboard design |
| Ndeti Melissa Mwende | Testing & debugging |
| Gitonga Benson Kamau | Documentation |

---

## 10. Links Summary

| Item | Link |
|---|---|
| Wokwi public simulation | https://wokwi.com/projects/468276398579020801 |
| GitHub repository | `https://github.com/<your-org>/<repo-name>` |
| Grafana dashboard | see `Grafana Dashboard.png` |
| InfluxDB data verification | see `Influx DB.png` |

---

## 11. References

- Wokwi Documentation — ESP32 Wi-Fi Networking: https://docs.wokwi.com/guides/esp32-wifi
- Wokwi Documentation — DHT22: https://docs.wokwi.com/parts/wokwi-dht22
- InfluxDB Client for Arduino (Tobias Schürg): https://github.com/tobiasschuerg/InfluxDB-Client-for-Arduino
- InfluxDB Cloud Documentation: https://docs.influxdata.com/influxdb/cloud/
- Grafana Cloud — InfluxDB Data Source: https://grafana.com/docs/grafana-cloud/connect-externally-hosted/data-sources/influxdb/
