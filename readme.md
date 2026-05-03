# Retail Analytics App

Edge AI system for retail analytics — person tracking, cross-camera re-identification, and loitering detection. Runs on NVIDIA Jetson hardware using DeepStream and TensorRT.

---

## Architecture

```
RTSP Cameras
    ↓
deepstream_service  — C++ DeepStream pipeline (detection, tracking, ReID)
    ↓ ZeroMQ
backend             — FastAPI (camera management, event storage, loitering rules)
    ↓
frontend            — Next.js dashboard
    ↓
nginx               — Reverse proxy (port 80)
```

See [PLATFORMS.md](PLATFORMS.md) for OpenVINO (x86) and RK3588 deployment variants.

---

## Quick Start — Jetson

### Prerequisites

- NVIDIA Jetson Orin NX / AGX / Xavier with JetPack 5.x
- Docker with nvidia runtime as default (`docker info | grep "Default Runtime"`)
- NGC account with API key — [ngc.nvidia.com](https://ngc.nvidia.com)

### 1. Clone

```bash
git clone --recurse-submodules git@github.com:atriva-ai/retail-analytics-app.git
cd retail-analytics-app
```

### 2. Download AI models (one-time per device)

Models are not stored in Git. Download them before building the Docker image. TRT engines are auto-generated from these ONNX files on first container start and written back to the same directory.

```bash
cd services/deepstream-jetson
export NGC_API_KEY=<your_key>
./scripts/download_peoplenet.sh   # → models/peoplenet/resnet34_peoplenet_int8.onnx
./scripts/download_reid.sh        # → models/reid/resnet50_market1501_aicity156.onnx
cd ../..
```

### 3. Configure environment

```bash
cp backend/.env.example backend/.env
# Edit backend/.env to set POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB
```

### 4. Build and start

```bash
docker compose -f docker-compose.jetson.yml up --build
```

First run takes **8–12 minutes** — Docker image builds (~5 min) then TRT engine generation (~3–5 min, one-time). Subsequent starts are ~30 seconds.

All 5 services are ready when you see:
```
deepstream_service  | ZMQ subscriber started
backend             | Application startup complete.
frontend            | ▲ Next.js ready
```

### 5. Register cameras and verify

```bash
# Register RTSP streams
curl -X POST "http://localhost:8001/cameras/register?camera_id=cam1&rtsp_url=rtsp://<host>:<port>/stream1"

# Check pipeline is live
curl http://localhost:8001/api/v1/video-pipeline/debug/
# → {"pipeline_alive": true, ...}

# Check detections are flowing
curl http://localhost:8001/shared/cameras/cam1/detections/latest

# Open dashboard
open http://localhost
```

For detailed DeepStream pipeline documentation, model configuration, and diagnostic tools, see [`services/deepstream-jetson/README.md`](services/deepstream-jetson/README.md).

---

## Project Structure

```
retail-analytics-app/
├── frontend/                     # Next.js dashboard
├── backend/                      # FastAPI — camera registry, detection store, loitering API
├── services/
│   └── deepstream-jetson/        # C++ DeepStream pipeline + Python FastAPI shim (Jetson)
├── nginx/
│   ├── conf.d/                   # Shared nginx config (OpenVINO / RK3588)
│   └── conf.d.jetson/            # Jetson-specific nginx config
├── docker-compose.jetson.yml     # Jetson full-stack compose
├── PLATFORMS.md                  # Multi-platform build and deployment guide
└── docs/test-reports/            # Test reports
```

---

## Stopping and restarting

```bash
# Stop (keeps volumes/data)
docker compose -f docker-compose.jetson.yml down

# Restart without rebuild (uses cached TRT engines — fast)
docker compose -f docker-compose.jetson.yml up

# Full rebuild (needed after code changes)
docker compose -f docker-compose.jetson.yml up --build
```

---

## Environment variables

| Variable | Service | Default | Purpose |
|---|---|---|---|
| `POSTGRES_USER` | db, backend | `atriva` | Database user |
| `POSTGRES_PASSWORD` | db, backend | — | Database password (set in `backend/.env`) |
| `POSTGRES_DB` | db, backend | `atrv-retail` | Database name |
| `CAMERAS_JSON` | deepstream_service | — | Pre-seed cameras: `{"cam1":"rtsp://..."}` |
| `NGC_API_KEY` | host (download scripts) | — | NGC model download |
