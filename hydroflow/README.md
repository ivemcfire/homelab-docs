HydroFlow
HydroFlow is an intelligent irrigation system that monitors water levels and automates refills using a Kubernetes-managed backend and ESP32-S2 Mini sensors.

🏗 Project Structure
The project is organized as a Monorepo to keep the hardware, software, and infrastructure in sync.

/firmware: C++ code for the ESP32-S2 Mini sensors (PlatformIO).

/backend: Node.js service that processes data and manages logic.

/frontend: Angular dashboard to visualize water levels in real-time.

/infra: Kubernetes manifests and configuration files for the cluster.

🚀 Quick Start

1. Hardware (The "Finger")
   The sensor is an ESP32-S2 Mini connected to a water level switch on Pin 7.

Location: firmware/tank-a-s2-mini

Action: Flash via PlatformIO. It uses a static IP (192.168.100.240) to communicate with the broker.

2. Infrastructure (The "Nervous System")
   The system runs on a k3s cluster.

MQTT Broker: Mosquitto (External IP: 192.168.100.207)

Database: PostgreSQL

Action: Deploy the manifests in the infra/ folder using kubectl apply -f.

3. Backend & Frontend
   The "Brain" and "Face" of the project.

Backend: Watches for MQTT messages on hydroflow/tank_A/level_low and saves them to Postgres.

Frontend: Displays the live status of your tanks.

📊 Data Flow
Sensor detects a change → Publishes to MQTT.

MQTT Broker passes data to the Backend.

Backend saves the event to PostgreSQL.

Frontend pulls the latest data to show on your screen.

🛠 Current Progress
[x] k3s Cluster setup with Mosquitto and Postgres.

[x] S2 Mini firmware with software debouncing.

[x] Successful data recording for Tank A.

[ ] Next Step: Implement automated refill logic (Refill Alert).
\
