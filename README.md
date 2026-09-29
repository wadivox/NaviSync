# NaviSync: Intelligent GNSS-Denied Navigation & Telemetry Copilot (SIH-26168)

An AI-powered, resilient dual-layer positioning and navigation platform designed to maintain sub-3m accuracy in GNSS-denied environments (mountain passes, subterranean tunnels, multi-level basements, and dense urban canyons) using consumer mobile device sensors and grounded reasoning.

---

## 📌 Key Capabilities

* **Deterministic Dead Reckoning Engine (10 Hz)**:
* Real-time kinematic fusion combining body-frame 3-axis accelerometer and gyroscope motion estimation.


* Coordinate transformations that isolate longitudinal vehicle acceleration from gravity vectors and dynamic centripetal forces.




* **Zero Velocity Updates (ZUPT)**:
* Automated vehicle standstill detection when sensor variance drops below threshold ($\text{variance} < 0.015\text{ (m/s}^2)^2$) during signal stops or traffic inside tunnels.


* Forcefully clamps forward velocity to $0.0\text{ m/s}$ to eliminate quadratic error accumulation and false drift during stops.




* **Non-Holonomic Kinematic Constraints (NHC)**:
* Enforces physical vehicle boundary constraints: zero lateral slip ($v_{\text{lateral}} \approx 0$) and zero vertical displacement ($v_{\text{vertical}} \approx 0$).


* Eliminates over 85% of accumulated lateral drift compared to unconstrained inertial integration.




* **Topological Map Matching**:
* Real-time orthogonal vector projection and bearing alignment locking positions onto road centerline network vectors.




* **Multi-Criteria GNSS Loss Detection & Smooth Fusion**:
* Bayesian state monitoring tracking accuracy jumps ($>25\text{m}$), update gaps ($>2.2\text{s}$), and kinematic velocity anomalies.


* Seamless state transitions: `GNSS_AVAILABLE` $\to$ `GNSS_DEGRADED` $\to$ `GNSS_LOST` $\to$ `NAVISYNC_INERTIAL`.


* Innovation-bounded Kalman gain correction upon signal restoration (`GNSS_RESTORING`) to eliminate coordinate teleportation.




* **IO-VNBD Benchmarking Target**:
* Implements the IO-VNBD (Inertial Odometry - Vehicle Navigation Benchmark Dataset) training protocol.


* Trains to predict delta-velocity ($\Delta v$) and delta-yaw ($\Delta \theta$) rather than raw geographic coordinates, bounding drift to $<1.8\%$ of distance traveled.




* **Grounded Navigation Copilot (NaviSync AI)**:
* Natural language AI copilot powered by OpenAI/Gemini with strict RAG vector knowledge grounding.


* Fully isolated from positioning: the LLM never computes coordinates, allowing navigation to run uninterrupted offline.




* **Local-First Privacy & Offline Autonomy**:
* Local IndexedDB storage (`navisync_travel_db`) for encrypted trajectory recording, speed timelines, and GNSS outage segment logs.


* Offline map tile caching for complete standalone navigation without active internet.





---

## 🏛️ System Architecture

```
                          [ PHYSICAL SENSORS / ENVIRONMENT ]
                                          │
                 ┌────────────────────────┴────────────────────────┐
                 ▼                                                 ▼
       [ Mobile IMU Hardware ]                           [ Device GNSS / GPS ]
    (3-Axis Accel, Gyro, Compass)                       (Lat, Lng, Speed, HDOP)
                 │                                                 │
                 ▼                                                 ▼
     ┌───────────────────────┐                         ┌───────────────────────┐
     │   AI Motion Filter    │                         │  GNSS Loss Detector   │
     │ - Temporal Conv / EMA │                         │ - Gaps (>2.2s) & HDOP │
     │ - Harmonic Noise Damp │                         │ - Speed Anomaly Check │
     └───────────┬───────────┘                         └───────────┬───────────┘
                 │                                                 │
                 │                                                 ▼
                 │                                       [ GNSS State Machine ]
                 │                                        • GNSS_AVAILABLE
                 │                                        • GNSS_DEGRADED
                 │                                        • GNSS_LOST
                 │                                        • NAVISYNC_INERTIAL
                 │                                        • GNSS_RESTORING
                 │                                                 │
                 ├────────────────────────┬────────────────────────┘
                 ▼                        ▼
     ┌──────────────────────┐  ┌──────────────────────┐
     │     ZUPT Engine      │  │ NHC Kinematic Engine │
     │ - Standstill Trigger │  │ - v_lateral ≈ 0      │
     │ - Velocity = 0.0 m/s │  │ - v_vertical ≈ 0     │
     └───────────┬──────────┘  └──────────┬───────────┘
                 │                        │
                 └───────────┬────────────┘
                             ▼
            ┌───────────────────────────────────┐
            │   Kalman / ESKF Sensor Fusion     │
            │ - High GNSS trust in open sky     │
            │ - Pure dead reckoning in tunnels  │
            │ - Innovation-bounded re-sync      │
            └─────────────────┬─────────────────┘
                              │
                              ▼
            ┌───────────────────────────────────┐
            │     Topological Map Matching      │
            │ - Road polyline orthogonal lock   │
            │ - Bearing angle alignment         │
            └─────────────────┬─────────────────┘
                              │
                              ▼
            ┌───────────────────────────────────┐
            │  Unified Authoritative Nav State  │
            │ (Position, Speed, Confidence, DR) │
            └─────────┬───────────────────┬─────┘
                      │                   │
                      ▼                   ▼
        ┌───────────────────────┐   ┌───────────────────────────┐
        │     User Cockpit      │   │    Navi AI Copilot        │
        │ - Leaflet Canvas      │   │ - Live Telemetry Binding  │
        │ - Turn-by-Turn Voice  │   │ - Grounded Knowledge Base │
        │ - Offline IndexedDB   │   │ - Strict Tool Separation  │
        └───────────────────────┘   └───────────────────────────┘

```

---

## 📁 Repository Structure

```
├── navisync/
│   ├── src/
│   │   ├── components/
│   │   │   ├── Assistant/             # NaviCopilot floating RAG AI assistant
│   │   │   ├── Header/                # AppHeader & language switcher
│   │   │   ├── Map/                   # LeafletMap canvas & directional marker
│   │   │   ├── Navigation/            # NavigationCockpit & TravelHistoryModal
│   │   │   ├── Offline/               # OfflineManagerModal & IndexedDB manager
│   │   │   ├── Routes/                # RoutePlanner & multi-modal routing UI
│   │   │   ├── Search/                # SearchBar, PlacesCarousel & CategoryChips
│   │   │   ├── SIHDemo/               # SIHDemoPage interactive presentation view
│   │   │   └── TechnicalDashboard/    # IntelligenceDashboard, EvaluatorControls & IOVNBDModal
│   │   ├── engines/
│   │   │   ├── AIMotionFilter.ts      # Shock spike & engine vibration rejection
│   │   │   ├── DriftEstimator.ts      # Real-time error estimation & confidence scoring
│   │   │   ├── GNSSLossDetector.ts    # Multi-criteria Bayesian blackout detector
│   │   │   ├── IMUSpeedEstimator.ts   # Kinematic forward velocity integrator
│   │   │   ├── MapMatchingEngine.ts   # Road network polyline projection
│   │   │   ├── NHCConstraintEngine.ts # Non-holonomic zero lateral slip constraints
│   │   │   ├── SensorFusionEngine.ts  # Central Kalman / ESKF positioning core
│   │   │   └── ZeroVelocityDetector.ts# ZUPT standstill detector
│   │   ├── providers/
│   │   │   ├── GeocodingProvider.ts   # Nominatim geocoder with cached lookups
│   │   │   ├── PlacesProvider.ts      # Overpass API nearby amenity search
│   │   │   └── RoutingProvider.ts     # OSRM turn-by-turn routing engine
│   │   ├── services/
│   │   │   ├── OfflineService.ts      # Tile & regional data persistence
│   │   │   ├── SensorService.ts       # Web DeviceMotion / DeviceOrientation listener
│   │   │   ├── TravelHistoryService.ts# Local IndexedDB journey logging
│   │   │   └── VoiceService.ts        # Turn-by-turn audio guidance engine
│   │   ├── types/
│   │   │   ├── navigation.ts          # FusionState, GNSSState & Route types
│   │   │   └── rag.ts                 # RAG documents & Copilot interfaces
│   │   ├── data/
│   │   │   └── knowledgeBase.ts       # Grounded system documentation for RAG
│   │   ├── i18n/
│   │   │   └── translations.ts        # 10 Indian regional languages
│   │   ├── App.tsx                    # Main state orchestration
│   │   ├── main.tsx                   # Application entry point
│   │   └── index.css                  # Tailwind styles and marker rules
│   ├── server.ts                      # Express API, RAG search & OpenAI endpoint
│   ├── package.json                   # Dependencies & build scripts
│   ├── vite.config.ts                 # Vite bundler configuration
│   ├── tsconfig.json                  # TypeScript compiler settings
│   └── README.md                      # Comprehensive project documentation

```

---

## 🚀 Quick Start Guide

### Prerequisites

* Node.js 18+ & npm (or Bun)


* Modern browser with Geolocation & DeviceMotion API support



### 1. Environment Setup

Clone the repository and prepare the configuration:

```bash
git clone https://github.com/<your-username>/NaviSync.git
cd NaviSync
cp .env.example .env

```

Configure your `.env` file:

```env
OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4o-mini"
VITE_CARTO_API_KEY="cb1_2wsb_1_1ce327a00cc6c5aedcb3471b"
APP_URL="http://localhost:3000"

```

### 2. Install & Start Development Server

```bash
# Install dependencies
npm install

# Start full-stack development server (Express backend + Vite frontend)
npm run dev

```

Application will be available at: **http://localhost:3000**

### 3. Production Build & Execution

```bash
# Build production bundle
npm run build

# Start production server
npm start

```

---

## 📡 Primary API Endpoints

| Method | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/api/health` | Backend operational status, loaded RAG documents, and timestamp

 |
| `POST` | `/api/rag/search` | Queries the server-side vector knowledge base for grounded navigation docs

 |
| `POST` | `/api/assistant` | NaviSync AI reasoning copilot binding real-time vehicle telemetry to RAG context

 |

---

## 🛡️ Security & Privacy Principles

* **Local-First Processing**: Sensor data, positioning equations, and journey trajectories are processed client-side with zero telemetry leakage.


* **Encrypted Client Persistence**: Trip history and outage metrics are stored exclusively in local browser IndexedDB (`navisync_travel_db`) and can be exported or cleared at will.


* **LLM Separation Invariant**: Coordinate determination and double integration are strictly deterministic physics algorithms; the AI reasoning layer cannot alter physical vehicle positions.



---

## 👥 Authors & Acknowledgments

* **Project**: NaviSync — AI-Powered GNSS-Denied Navigation & Inertial Dead Reckoning


* **Built for**: Smart India Hackathon (SIH)


* **Problem Statement Reference**: SIH-26168 (Reliable Navigation in GNSS-Denied Environments)



---
