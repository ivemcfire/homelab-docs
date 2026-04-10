Project Context: HydroFlow IoT System

1. Core Architecture
   Orchestration: k3s cluster (Namespace: hydroflow) running on ARM64 nodes (one6t, one61, one62) and k3master.

Messaging: Mosquitto MQTT (LoadBalancer: 192.168.100.207:1883).

Database: PostgreSQL (Internal cluster service).

Registry: Private Gitea at http://192.168.100.206.

Hardware: ESP32-S2 Mini (Static IP: 192.168.100.240, Pin 7 Water Sensor).

2. Directory Structure (Monorepo Rules)
   Maintain this strict separation of concerns to avoid project bloat:

/backend: Node.js service. Consumes MQTT, writes to Postgres.

/firmware: Hardware-specific code. Organized by node (e.g., /firmware/tank-a-s2-mini). Contains platformio.ini and src/.

/frontend: Angular Dashboard. Strictly web files only (no C++ or K8s YAMLs).

/infra: Kubernetes manifests (.yaml), Dockerfiles, and config files (mosquitto.conf).

3. Current State (March 2026)
   Telemetry: Tank A sensor is live. It publishes to hydroflow/tank_A/level_low.

Firmware Logic: Uses INPUT_PULLUP on Pin 7. Implements a 500ms non-blocking software debounce to prevent "bouncy" database entries.

Data Mapping: The backend parses the MQTT topic to use the second segment as the sensor_id (e.g., tank_A).

Database Status: Verified. Table sensor_readings is capturing tank_A status (ON/OFF) with server-side timestamps.

4. Operating Instructions for Gemini
   Cleanliness: Always ensure node_modules, .pio, and .angular folders are ignored.

Hardware Changes: If modifying firmware, respect the native USB CDC flags for the S2 Mini in platformio.ini.

Kubernetes: All infrastructure changes must be reflected in the /infra folder manifests.

Logic Phase (Next): Implement "Refill Alert" logic in the backend. If tank_A is ON (Low) for >30s, trigger a command to a (future) valve node.
