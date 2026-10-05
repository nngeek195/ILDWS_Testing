# Intelligent Landslide Detection and Warning System

An end-to-end landslide monitoring and early-warning platform designed for harsh outdoor environments and unreliable network connectivity.

## 1. How the Landslide Detection System Works

The system uses a three-layer architecture:

1. **Sensor layer** – Solar-powered ESP32 nodes collect geotechnical measurements on high-risk slopes.
2. **Edge layer** – A Raspberry Pi gateway receives LoRaWAN telemetry, performs local inference, buffers data, and triggers local alarms when necessary.
3. **Cloud layer** – GCP services process, store, analyze, and display data for emergency operators.

### System Architecture

```text
[ MONITORED SLOPE ]
      │
      ├── ESP32 Node 1 ──┐
      ├── ESP32 Node 2 ──┼──► [ LoRaWAN Radio ] ──► [ Edge Gateway: Raspberry Pi ]
      └── ESP32 Node N ──┘                              │
                                                       ├── SQLite offline buffer
                                                       ├── TFLite local inference
                                                       └── Local siren relay (offline-safe)
                                                       │
                                                       ▼
                                             [ 4G/LTE or Satellite ]
                                                       │
                                                       ▼
                                                  [ CLOUD (GCP) ]
                                                       ├── Mosquitto MQTT broker
                                                       ├── ETL processor
                                                       ├── TimescaleDB time-series store
                                                       ├── Cloud ML: LSTM predictor
                                                       ├── FastAPI backend and WebSockets
                                                       └── Notification service: SMS and FCM push
                                                       │
                                                       ▼
                                             [ EMERGENCY DASHBOARD ]
                                                       ├── Live Leaflet slope map
                                                       └── Pre-failure alert banners
```

### The End-to-End Detection Chain

#### 1. Geotechnical Ground Sensing — Sensor Nodes

Autonomous, solar-powered ESP32 microcontrollers are placed directly on high-risk slopes. Each node measures four key landslide precursors:

- **Soil moisture and saturation:** Water accumulation decreases soil shear strength.
- **Tilt and inclination:** Detects micro-movements and slope displacement on the X and Y axes.
- **Cumulative rainfall:** Identifies intense precipitation thresholds.
- **Ground vibration and seismicity:** Detects shearing and subsurface fractures.

> **Note:** Sensor nodes use manually surveyed GPS coordinates stored in the database instead of onboard GPS hardware to reduce battery consumption and cost.

#### 2. Long-Range LoRaWAN Transmission

Sensor nodes transmit compact telemetry packets over LoRaWAN. This provides kilometer-scale coverage across rugged, forested mountain terrain without requiring Wi-Fi or cellular coverage at every sensor location.

#### 3. Edge Gateway and Offline Failover — Raspberry Pi

The Raspberry Pi gateway is installed at the monitoring site and receives LoRaWAN packets.

- **Local TFLite inference:** Runs a quantized neural network directly on the Raspberry Pi. If slope movement occurs while the cellular link is unavailable, the gateway can activate physical on-site sirens immediately.
- **SQLite buffering:** Caches sensor data during internet outages and synchronizes the buffered records with the cloud when connectivity is restored.

#### 4. Cloud Pipeline and TimescaleDB — GCP

The cloud pipeline processes and stores incoming telemetry:

- **Mosquitto broker:** Ingests high-frequency sensor readings.
- **ETL processor:** Validates sensor bounds, filters sensor noise, checks battery levels, and inserts records into a TimescaleDB hypertable.
- **Data retention:** Maintains data for a minimum of seven years for forensic geological analysis and government compliance.

#### 5. Deep-Learning Risk Scoring — LSTM Neural Network

A two-layer Long Short-Term Memory (LSTM) network processes a sliding window of historical time-series data and produces a continuous landslide risk score from `0.00` to `1.00`.

| Risk level | Score range | Action triggered |
|---|---:|---|
| 🟢 **Green** | `0.00–0.39` | Normal conditions; baseline telemetry rate. |
| 🟡 **Yellow** | `0.40–0.64` | Elevated risk; sampling frequency increases automatically. |
| 🟠 **Orange** | `0.65–0.84` | High probability; advisory alerts are sent to disaster agencies and evacuation preparations begin. |
| 🔴 **Red** | `0.85–1.00` | Imminent slope failure; public sirens activate and emergency SMS and push notifications are broadcast. |

#### 6. Alert Dispatch and Operator Dashboard

- **Notification service:** Sends SMS notifications through Twilio and push notifications through Firebase Cloud Messaging (FCM). Redis is used to deduplicate alerts and reduce alarm fatigue.
- **Web dashboard:** Built with React 18, Tailwind CSS, and Leaflet. Emergency operators can view colored risk polygons on topographic maps, inspect real-time telemetry charts, and acknowledge incidents with one click.

## 2. Current Project Status

Based on the repository implementation, test suites, and simulation tooling, the software platform is approximately **85–90% complete** and fully testable in simulation.

### Completed and Working

| Subsystem | Status | Details |
|---|---|---|
| Architecture and specifications | ✅ Completed | SRS, SDD, API specifications, and database schemas are available in the repository documentation. |
| Cloud microservices | ✅ Completed | FastAPI REST/WebSocket server, ETL pipeline, notification service, and ML inference service. |
| Edge gateway processor | ✅ Completed | SQLite store-and-forward buffering, network failover monitoring, and local alert triggering. |
| Machine-learning models | ✅ Completed | LSTM model, autoencoder anomaly detector, training script with MLflow tracking, and TFLite export. |
| Web dashboard | ✅ Completed | React 18, Vite, Leaflet map view, real-time WebSocket telemetry charts, and alert management views. |
| DevOps and cloud IaC | ✅ Completed | Terraform scripts for GCP infrastructure, Kubernetes GKE manifests, Dockerfiles, and `docker-compose.yml`. |
| End-to-end simulation | ✅ Completed | `simulate_landslide_event.py` simulates the complete GREEN → RED event and validates alert triggers. |
| Integration test suite | ✅ Completed | Full-pipeline tests in `test_full_pipeline.py` and edge offline-failover tests in `test_edge_failover.py`. |

### In Progress and Remaining Work

1. **Physical ESP32 firmware implementation**
   - The firmware entry point, `main.cpp`, currently contains code stubs.
   - Low-level C++ drivers still need to be implemented for the physical hardware, including:
     - I2C/SPI communication with MPU6050 or ADXL345 accelerometers.
     - ADC calibration for soil-moisture probes.
     - Pulse counting for rain gauges.
     - SX1276 LoRaWAN deep-sleep and power management.

2. **Real-world geotechnical data calibration**
   - The ML models are structured and functional, but they require training with real soil characteristics, including Soil Water Characteristic Curves (SWCC), and historical landslide records from the target deployment region.

3. **Physical field testing**
   - Prototype nodes must be deployed on an actual hill or slope to verify LoRa signal attenuation through soil, trees, and severe rainstorms.

## 3. Local Testing

The entire software pipeline can be tested locally without physical hardware.

### Prerequisites

- Docker and Docker Compose
- Python 3.x
- A browser for the dashboard

### Start the System

1. Start all containers:

   ```bash
   docker compose up -d --build
   ```

2. Apply database migrations:

   ```bash
   docker compose exec fastapi-backend alembic upgrade head
   ```

3. Open the web dashboard at:

   ```text
   http://localhost:3080
   ```

4. Trigger a simulated landslide event:

   ```bash
   python scripts/simulate_landslide_event.py \
     --slope-id SLP-001 \
     --duration-minutes 5 \
     --speed-multiplier 2
   ```

The dashboard should show soil moisture, rainfall, tilt, and vibration increasing in real time. The system should progress through the following states:

```text
GREEN → YELLOW → ORANGE → RED
```

Alerts should be triggered as the simulated risk level increases.
