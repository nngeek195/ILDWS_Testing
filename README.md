 ## 1. How the Landslide Detection System Works                                                                                                                              
                                                                                                                                                                              
  The system is structured into a 3-layer hierarchy designed to handle harsh outdoor hill conditions and poor network connectivity:                                           
                                                                                                                                                                              
    [ MONITORED SLOPE ]                                                                                                                                                       
      │                                                                                                                                                                       
      ├─ ESP32 Node 1 ──┐                                                                                                                                                     
      ├─ ESP32 Node 2 ──┼─► [LoRaWAN Radio] ──► [ Edge Gateway: Raspberry Pi ]                                                                                                
      └─ ESP32 Node N ──┘                         │  • SQLite Offline Buffer                                                                                                  
                                                  │  • TFLite Local Inference                                                                                                 
                                                  │  • Local Siren Relay (Offline Safe!)                                                                                      
                                                  │                                                                                                                           
                                                  ▼ (4G/LTE / Satellite)                                                                                                      
                                          [ CLOUD (GCP) ]                                                                                                                     
                                          ├─ Mosquitto MQTT Broker                                                                                                            
                                          ├─ ETL Processor                                                                                                                    
                                          ├─ TimescaleDB (7-year time-series store)                                                                                           
                                          ├─ Cloud ML (LSTM 2-hr predictor)                                                                                                   
                                          ├─ FastAPI Backend + WebSockets                                                                                                     
                                          └─ Notification Service (SMS / FCM Push)                                                                                            
                                                  │                                                                                                                           
                                                  ▼                                                                                                                           
                                       [ EMERGENCY DASHBOARD ]                                                                                                                
                                        • Live Leaflet Slope Map                                                                                                              
                                        • Pre-failure Alert Banners                                                                                                           
                                                                                                                                                                              
  ### The End-to-End Detection Chain:                                                                                                                                         
                                                                                                                                                                              
  1. Geotechnical Ground Sensing (Layer 1 - Sensor Nodes):                                                                                                                    
      • Autonomous, solar-powered ESP32 microcontrollers are placed directly on high-risk slopes.                                                                             
      • They measure 4 key landslide precursors:                                                                                                                              
          • Soil Moisture / Saturation: Water buildup decreases soil shear strength.                                                                                          
          • Tilt / Inclinometer (X/Y axis): Micro-movements and slope displacement.                                                                                           
          • Cumulative Rainfall: Identifies intense precipitation thresholds.                                                                                                 
          • Ground Vibration / Seismicity: Detects shearing and subsurface fractures.                                                                                         
      • Note: Sensor nodes use manually surveyed GPS coordinates stored in the database instead of onboard GPS hardware to save battery and cost.                             
  2. Long-Range LoRaWAN Transmission:                                                                                                                                         
      • Nodes transmit compact telemetry packets via LoRaWAN radio (sub-GHz) to cover kilometers of rugged, forested mountain terrain without needing WiFi or cellular        
      coverage at every sensor.                                                                                                                                               
  3. Edge Gateway & Offline Failover (Layer 2 - Raspberry Pi):                                                                                                                
      • Located at the slope site. Receives LoRa packets.                                                                                                                     
      • Local TFLite Inference: Runs a quantized neural network directly on the Raspberry Pi. If a sudden slope movement occurs and the cellular internet link is severed by a
      storm, the gateway can directly sound physical on-site sirens to evacuate the area immediately.                                                                         
      • SQLite Buffer: Caches all sensor data locally during internet blackouts and automatically syncs in batches to the cloud once network connectivity is restored.        
  4. Cloud Pipeline & TimescaleDB (Layer 3 - GCP):                                                                                                                            
      • Mosquitto Broker: Ingests high-frequency sensor readings.                                                                                                             
      • ETL Processor: Validates sensor bounds, filters sensor noise, checks battery levels, and inserts records into a TimescaleDB hypertable.                               
      • Data is retained for minimum 7 years for forensic geological analysis and government compliance.                                                                      
  5. Deep Learning Risk Scoring (LSTM Neural Network):                                                                                                                        
      • A 2-layer LSTM (Long Short-Term Memory) network processes a sliding window of historical time-series data.                                                            
      • It outputs a continuous Landslide Risk Score (0.00 to 1.00) categorized into 4 operational tiers:                                                                     
                                                                                                                                                                              
   Risk Level                                   │ Score Range                                  │ Action Triggered
  ──────────────────────────────────────────────┼──────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────
   🟢 GREEN                                     │ 0.00 - 0.39                                  │ Normal conditions; baseline telemetry rate
   🟡 YELLOW                                    │ 0.40 - 0.64                                  │ Elevated risk; sampling frequency automatically increases
   🟠 ORANGE                                    │ 0.65 - 0.84                                  │ High probability; advisory alerts to disaster agencies, prepare evacuation
   🔴 RED                                       │ 0.85 - 1.00                                  │ Imminent slope failure; public sirens sound, emergency SMS & push broadcasts
                                                                                                                                                                              
  6. Alert Dispatch & Operator Dashboard:                                                                                                                                     
      • Notification Service: Dispatches SMS via Twilio and push notifications via Firebase (FCM). Uses Redis to deduplicate alerts and prevent "alarm fatigue".              
      • Web Dashboard: Built with React 18, Tailwind, and Leaflet maps. Emergency operators see colored risk polygons on topographic maps, real-time telemetry graphs, and can
      acknowledge incidents with 1 click.                                                                                                                                     
                                                                                                                                                                              
  ──────                                                                                                                                                                      
  ## 2. How the Project Plan Is Going (Current Status)                                                                                                                        
                                                                                                                                                                              
  Checking the git commit history, codebase implementation, and test suites, the software platform is ~85% to 90% completed and fully testable in simulation.                 
                                                                                                                                                                              
  ### What is Already Built & Working:                                                                                                                                        
                                                                                                                                                                              
   Subsystem                     │ Status                │ Details
  ───────────────────────────────┼───────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   Architecture & Specifications │ ✅ Completed          │ Comprehensive SRS, SDD, API specifications, and Database schemas in .
   Cloud Microservices           │ ✅ Completed          │ FastAPI REST/WebSocket server, ETL pipeline, Notification service, and ML inference service in .
   Edge Gateway Processor        │ ✅ Completed          │ SQLite store-and-forward buffering (buffer.py), network failover monitor, and local alert trigger.
   Machine Learning Models       │ ✅ Completed          │ LSTM model (lstm_model.py), Autoencoder anomaly detector, training script with MLflow tracking, and TFLite export.
   Web Dashboard                 │ ✅ Completed          │ React 18 + Vite + Leaflet map view (), real-time WebSocket telemetry charts, and alert management views.
   DevOps & Cloud IaC            │ ✅ Completed          │ Terraform scripts for GCP infrastructure (), Kubernetes GKE manifests, Dockerfiles, and docker-compose.yml.
   End-to-End Simulation         │ ✅ Completed          │ simulate_landslide_event.py simulates the entire GREEN → RED landslide event and validates alert triggers.
   Integration Test Suite        │ ✅ Completed          │ Full pipeline tests (test_full_pipeline.py) and edge offline failover tests (test_edge_failover.py).
  ──────                                                                                                                                                                      
  ### What Is Still In Progress / What Remains to Be Done:                                                                                                                    
                                                                                                                                                                              
  1. Physical Hardware ESP32 Firmware Implementation:                                                                                                                         
      • The firmware entry point (main.cpp) currently contains code stubs.                                                                                                    
      • Specific low-level C++ drivers for the physical hardware need to be written (e.g., reading I2C/SPI from MPU6050/ADXL345 accelerometers, ADC calibration for soil      
      moisture probes, pulse counting for rain gauges, and SX1276 LoRaWAN deep-sleep power management).                                                                       
  2. Real-World Geotechnical Data Calibration:                                                                                                                                
      • The ML models are structured and functional, but need training on real soil characteristics (Soil Water Characteristic Curves - SWCC) and historical landslide event  
      records from the target deployment region.                                                                                                                              
  3. Physical Field Testing:                                                                                                                                                  
      • Deploying physical prototype nodes on an actual hill/slope to verify LoRa signal attenuation through soil, trees, and severe rainstorms.
  
  ──────
  ### How You Can Test the System Right Now:
  
  You can test the entire pipeline locally without physical hardware:
  
  1. Spin up all containers:
    docker compose up -d --build
  
  2. Apply database migrations:
    docker compose exec fastapi-backend alembic upgrade head
  
  3. Open the web dashboard:
  Visit http://localhost:3080 in your browser.
  4. Trigger a simulated landslide event:
    python scripts/simulate_landslide_event.py --slope-id SLP-001 --duration-minutes 5 --speed-multiplier 2
  You will see the soil moisture, rainfall, tilt, and vibration escalate on the dashboard in real-time, watching the system progress from GREEN → YELLOW → ORANGE → RED,      
  triggering alerts along the way.
