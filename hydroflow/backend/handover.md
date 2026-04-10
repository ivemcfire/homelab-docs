
Hello Claude,

Here is the handover for the HydroFlow IoT project. My task was to get the project ready for you to deploy on a local k3s cluster.

## 1. Project Overview

HydroFlow is an IoT system for monitoring and controlling irrigation. It consists of hardware nodes (ESP32), a messaging layer (MQTT), a backend service to process data, a database to store it, and a web frontend for visualization and control.

## 2. System Architecture

*   **Orchestration**: k3s running on ARM64 nodes. The target namespace is `hydroflow`.
*   **Messaging**: Mosquitto MQTT broker. The `CONTEXT.md` mentions a LoadBalancer at `192.168.100.207:1883`.
*   **Database**: PostgreSQL.
*   **Backend**: A Node.js service that subscribes to MQTT topics, parses sensor data, and stores it in the PostgreSQL database.
*   **Frontend**: An Angular application with Server-Side Rendering (SSR) for displaying data and interacting with the system.

## 3. Codebase Structure (Monorepo)

*   `c:/Users/Work/Documents/Projects/hydroflow/backend`: The Node.js backend.
    *   **Entrypoint**: `src/index.js`.
    *   **Dependencies**: `package.json`.
    *   **Dockerfile**: A multi-stage Dockerfile exists at `backend/Dockerfile`. It builds a production-ready ARM64 image based on `node:20-slim`. It exposes port 3000 and includes a health check.
*   `c:/Users/Work/Documents/Projects/hydroflow/frontend`: The Angular SSR frontend.
    *   **Dependencies**: `package.json`.
    *   **Dockerfile**: **Does not exist.** You will need to create one. A typical multi-stage Dockerfile for an Angular SSR app would be appropriate.
        *   **Build Stage**: Use a Node image (e.g., `node:20`) to install dependencies (`npm install`) and build the app (`npm run build`).
        *   **Run Stage**: Use a slim Node image (e.g., `node:20-slim`), copy the built artifacts from the build stage (from `dist/app/server/`), install *only* production dependencies (`npm install --omit=dev`), and run the server (`node server.mjs`). The server port is likely configured in `src/server.ts` or as an environment variable (the backend uses port 3000, so the frontend might use another, e.g. 4000).
*   `c:/Users/Work/Documents/Projects/hydroflow/firmware`: Contains code for the ESP32 hardware nodes. Not relevant for the k3s deployment itself.
*   `c:/Users/Work/Documents/Projects/hydroflow/infra`: This directory is **empty**. It is intended to hold the Kubernetes manifests. You will need to create them.

## 4. Deployment Requirements & Steps

### 4.1. Containerization

1.  **Backend Image**: Build the Docker image from `backend/Dockerfile`. The target architecture is `linux/arm64`. Tag it and push it to a container registry. The project context mentions a private Gitea registry at `192.168.100.206`.
2.  **Frontend Image**: Create the `Dockerfile` for the frontend as described above. Build, tag, and push the `linux/arm64` image to the registry.

### 4.2. Kubernetes Manifests (To Be Created in `/infra`)

You need to create the following Kubernetes objects for the `hydroflow` namespace:

*   **Namespace**: `00-namespace.yaml` to create the `hydroflow` namespace.
*   **PostgreSQL**:
    *   A `Deployment` or `StatefulSet` for PostgreSQL. Use a standard image like `postgres:15-alpine`.
    *   A `Service` to expose PostgreSQL within the cluster (e.g., `postgres-svc`).
    *   A `PersistentVolume` and `PersistentVolumeClaim` to ensure data persistence.
    *   A `Secret` to manage the PostgreSQL username and password (`POSTGRES_USER`, `POSTGRES_PASSWORD`).
*   **Mosquitto**:
    *   A `Deployment` for the MQTT broker. Use an image like `eclipse-mosquitto`.
    *   A `Service` of type `LoadBalancer` to expose MQTT on port `1883`. The context suggests the IP `192.168.100.207` was used, which you can request from the k3s metallb LoadBalancer.
*   **Backend Deployment**:
    *   A `Deployment` for the backend service.
    *   Use the Docker image you built.
    *   Inject the database credentials and host/port as environment variables from the PostgreSQL secret and service.
        *   `DB_HOST`: `postgres-svc.hydroflow.svc.cluster.local`
        *   `DB_USER`, `DB_PASSWORD`: From the secret.
        *   `MQTT_HOST`: `mosquitto-svc.hydroflow.svc.cluster.local`
    *   A `Service` to expose the backend within the cluster.
*   **Frontend Deployment**:
    *   A `Deployment` for the frontend service.
    *   Use the Docker image you built.
    *   A `Service` (likely `LoadBalancer` or `NodePort`) to expose the frontend to users.
    *   The `GEMINI_API_KEY` needs to be provided. See the 'Recent Changes' section below.

## 5. Summary of My Recent Changes (Important Context)

I was tasked with getting the frontend running for testing. I made the following changes to fix build and configuration errors that arose from a recent project reorganization:

1.  **`angular.json`**: The `build.options.tsConfig` was pointing to a non-existent `tsconfig.app.json`. I updated it to point to `tsconfig.json`.
2.  **`tsconfig.json`**:
    *   It contained `references` to non-existent `tsconfig.app.json` and `tsconfig.spec.json`, which I removed.
    *   The TypeScript compiler was trying to compile test files (`.spec.ts`), causing errors. I added `"exclude": ["src/**/*.spec.ts"]` to `tsconfig.json` to fix this.
3.  **`package.json` (frontend)**:
    *   The `dev` script was complex, used invalid argument syntax for `ng serve`, and had shell-specific syntax that failed on Windows.
    *   I simplified the `dev` script to `cross-env ng serve --port=3000 --host=0.0.0.0`.
4.  **Gemini API Key**:
    *   To handle the `GEMINI_API_KEY` for the frontend, I updated `angular.json` to include a `define` block in the build options: `"define": { "GEMINI_API_KEY": "process.env.GEMINI_API_KEY" }`.
    *   This allows the key to be set via an environment variable at build time, which is a better practice than the previous command-line argument approach.
5.  **Model Update**: The user requested to switch the AI model. I updated `frontend/src/app/gemini.service.ts` to use `gemini-3.1-pro` instead of `gemini-3-flash-preview`.

This should give you all the information you need to proceed with the deployment. Good luck!
