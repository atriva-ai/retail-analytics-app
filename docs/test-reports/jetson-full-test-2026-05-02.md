# Jetson Full-Stack Test Report

**Date:** 2026-05-02  
**Branch:** `fix/deepstream-jetson-base-image`  
**Platform:** NVIDIA Jetson Orin NX — JetPack 5.x, kernel `5.10.104-tegra`  
**Docker runtime:** `nvidia` (default)

---

## Summary

| Category | Result |
|---|---|
| Python static analysis | ✅ Pass — 0 errors |
| Unit tests (deepstream-jetson API) | ✅ 39 / 39 passed |
| Backend Docker image build | ✅ Pass |
| deepstream-jetson Docker image build | ✅ Pass (C++ + Python) |
| deepstream_service API contract | ✅ All endpoints verified |
| Full stack (`docker-compose.jetson.yml`) | ✅ All 5 services healthy |
| Bug found and fixed | ⚠️ nginx Jetson config — see details |

---

## 1. Python Static Analysis

**Scope:** All `.py` files in `backend/app/`, `services/deepstream-jetson/api/`, `services/ai-inference/app/`  
**Tool:** `python3 -m py_compile`  
**Result:** ✅ All files compile without syntax errors.

---

## 2. Unit Tests — deepstream-jetson API Layer

**File:** `services/deepstream-jetson/tests/test_deepstream_api.py`  
**Tool:** `pytest 8.3.5` on Python 3.8.10  
**No Docker, GPU, or C++ binary required** — zmq stubbed, `RUNTIME_CONFIG` patched to `/tmp`.

```
============================= test session starts ==============================
platform linux -- Python 3.8.10, pytest-8.3.5, pluggy-1.5.0
collected 39 items

TestCameraRegistry::test_activate_deactivate             PASSED
TestCameraRegistry::test_cameras_json_env_preload        PASSED
TestCameraRegistry::test_cameras_json_invalid_does_not_crash PASSED
TestCameraRegistry::test_has_cameras                     PASSED
TestCameraRegistry::test_multiple_cameras_in_yaml        PASSED
TestCameraRegistry::test_register_and_list               PASSED
TestCameraRegistry::test_register_writes_yaml            PASSED
TestCameraRegistry::test_resolve_source_order            PASSED
TestCameraRegistry::test_restart_fn_called_on_register   PASSED
TestCameraRegistry::test_restart_fn_called_on_unregister PASSED
TestCameraRegistry::test_thread_safety_concurrent_register PASSED
TestCameraRegistry::test_unregister_missing_returns_false PASSED
TestCameraRegistry::test_unregister_removes_camera       PASSED
TestDetectionStore::test_cameras_list                    PASSED
TestDetectionStore::test_empty_detections                PASSED
TestDetectionStore::test_get_missing_returns_none        PASSED
TestDetectionStore::test_latest_overwritten_on_update    PASSED
TestDetectionStore::test_to_response_no_filter           PASSED
TestDetectionStore::test_to_response_with_filter         PASSED
TestDetectionStore::test_translate_bbox_conversion       PASSED
TestDetectionStore::test_update_and_get                  PASSED
TestRoutes::test_detections_no_pipeline_503              PASSED
TestRoutes::test_inference_status_no_camera              PASSED
TestRoutes::test_list_cameras_empty                      PASSED
TestRoutes::test_model_info                              PASSED
TestRoutes::test_models                                  PASSED
TestRoutes::test_register_and_unregister_camera          PASSED
TestRoutes::test_root                                    PASSED
TestRoutes::test_shared_cameras_empty                    PASSED
TestRoutes::test_stop_inference                          PASSED
TestRoutes::test_unregister_missing_camera_404           PASSED
TestRoutes::test_video_pipeline_debug                    PASSED
TestRoutes::test_video_pipeline_health                   PASSED
TestRoutes::test_vp_decode_start_no_url_no_registered_400 PASSED
TestRoutes::test_vp_decode_start_with_url                PASSED
TestRoutes::test_vp_decode_status_unknown_camera         PASSED
TestRoutes::test_vp_decode_stop                          PASSED
TestRoutes::test_vp_latest_frame_503                     PASSED
TestRoutes::test_vp_video_info_url                       PASSED

============================== 39 passed in 5.19s ==============================
```

### Test coverage by class

| Class | Tests | Coverage |
|---|---|---|
| `CameraRegistry` | 13 | register/unregister, YAML write, activate/deactivate, source index ordering, CAMERAS_JSON env preload, invalid JSON, restart callback, multi-camera YAML, thread safety (10 concurrent registers) |
| `DetectionStore` | 8 | update/get, missing key returns None, multi-camera tracking, `to_response` with/without filter, latest-frame overwrite, bbox `[l,t,l+w,t+h]` translation, empty detections |
| FastAPI Routes | 18 | root, model/info, models, cameras CRUD, VP health/debug, inference start/stop/status, VP decode start/stop/status, latest-frame 503, video-info-url stub, shared/cameras |

---

## 3. Docker Image Builds

### 3a. Backend (`retail-analytics-app-backend`)

**Base image:** `python:3.11-slim`  
**Result:** ✅ Built successfully  
**Packages installed:** fastapi, uvicorn[standard], sqlalchemy, psycopg2-binary, httpx, python-multipart, requests

### 3b. deepstream-jetson (`retail-analytics-app-deepstream_service`)

**Base image:** `nvcr.io/nvidia/deepstream-l4t:6.2-triton` (ARM64, JetPack 5.x)  
**Result:** ✅ Built successfully — full C++ compile + Python install  

**C++ build steps (from Dockerfile):**
- `cmake .. -DDS_SDK_PATH=/opt/nvidia/deepstream/deepstream` — CMake version required: **3.13** (lowered from 3.16 to match DeepStream L4T base image)
- `make -j$(nproc)` — compiled `edge_retail_core_engine` binary

**Python packages installed:**
- fastapi ≥0.110.0, uvicorn[standard] ≥0.29.0, pyzmq ≥25.0, numpy ≥1.21, pyyaml ≥6.0, **python-multipart ≥0.0.9** (added — required by FastAPI form endpoints)

### 3c. Frontend (`retail-analytics-app-frontend`)

**Base image:** `node:18-alpine`  
**Result:** ✅ Built successfully — Next.js 15.2.4 production build

```
Route (app)                          Size  First Load JS
┌ ○ /                             4.35 kB       231 kB
├ ○ /cameras                       438 B        119 kB
├ ○ /person-activity             7.84 kB        141 kB
├ ○ /product-analytics             216 B        253 kB
├ ○ /settings                    23.7 kB        163 kB
└ ○ /staff-monitoring            5.78 kB        263 kB
```

---

## 4. deepstream_service API Contract Verification

**Container:** `deepstream-retail-test` (standalone, no cameras)

| Endpoint | Method | Expected | Result |
|---|---|---|---|
| `/` | GET | `{architecture: deepstream}` | ✅ |
| `/model/info` | GET | `{accelerators: [tensorrt]}` | ✅ |
| `/models` | GET | `{available_models: [peoplenet+reid]}` | ✅ |
| `/cameras` | GET | `{cameras: []}` | ✅ |
| `/cameras/register` | POST | `{status: registered}` | ✅ |
| `/cameras/{id}` | DELETE | `{status: unregistered}` | ✅ |
| `/cameras/nonexistent` | DELETE | HTTP 404 | ✅ |
| `/inference/continuous/start` | POST | `{status: started, architecture: deepstream}` | ✅ |
| `/inference/continuous/stop` | POST | `{status: stopped}` | ✅ |
| `/inference/continuous/status` | GET | `{running: true/false}` | ✅ |
| `/shared/cameras` | GET | `{cameras: []}` | ✅ |
| `/shared/cameras/{id}/detections/latest` | GET | HTTP 503 (no pipeline) | ✅ |
| `/api/v1/video-pipeline/health/` | GET | `{status: ok, architecture: deepstream}` | ✅ |
| `/api/v1/video-pipeline/debug/` | GET | `{pipeline_alive: false}` | ✅ |
| `/api/v1/video-pipeline/decode/` | POST (form) | `{status: started}` | ✅ |
| `/api/v1/video-pipeline/decode/stop/` | POST (form) | `{status: stopped}` | ✅ |
| `/api/v1/video-pipeline/decode/status/` | GET | `{frame_count: 0/1}` | ✅ |
| `/api/v1/video-pipeline/latest-frame/` | GET | HTTP 503 (by design) | ✅ |
| `/api/v1/video-pipeline/video-info-url/` | POST (form) | H264 stub metadata | ✅ |

**Pipeline lifecycle observed in logs:**
```
ZMQ subscriber connected to tcp://localhost:5555
Registered camera cam1 -> rtsp://test/stream1
Restarting DeepStream pipeline
Launched core_engine PID 28          ← C++ binary starts
core_engine stopped                  ← exits (no real RTSP)
```
C++ binary launches correctly on camera registration. Exits because test URL `rtsp://test/stream1` is unreachable — expected in this context.

---

## 5. Full Stack Smoke Test (`docker-compose.jetson.yml`)

### Container status (all healthy)

| Service | Image | Status | Ports |
|---|---|---|---|
| `deepstream_service` | `deepstream-l4t:6.2-triton` | ✅ Up | 8001 |
| `backend` | `python:3.11-slim` | ✅ Up | 8000 |
| `db` | `postgres:15.5` | ✅ Up | 5432 |
| `frontend` | `node:18-alpine` | ✅ Up | 3000 |
| `nginx` | `nginx:1.25.3-alpine` | ✅ Up (after fix) | 80, 443 |

### Service health checks

| Check | Result |
|---|---|
| `GET :8001/` | ✅ `{architecture: deepstream}` |
| `GET :8001/api/v1/video-pipeline/health/` | ✅ `{status: ok}` |
| `GET :8000/health` | ✅ `{status: healthy, database: connected}` |
| `GET :8000/api/v1/ai-inference/health/` | ✅ Proxied → deepstream_service |
| `GET :8000/api/v1/video-pipeline/health/` | ✅ Proxied → deepstream_service |
| `GET :80/` (nginx → frontend) | ✅ HTTP 200 |
| `GET :80/health` | ✅ `healthy` |
| `GET :80/api/v1/cameras/` (nginx → backend) | ✅ HTTP 200 |
| `GET :80/ai-inference/model/info` (nginx debug) | ✅ TensorRT model info |
| `GET :80/video-pipeline/api/v1/video-pipeline/health/` (nginx debug) | ✅ `{status: ok}` |

---

## 6. Bug Found and Fixed — nginx Jetson Config

### Root cause

`nginx/conf.d/default.conf` (shared across all platforms) contains two upstream references that don't exist on Jetson:

```nginx
location /video-pipeline/ {
    proxy_pass http://video_pipeline:8002/;   # ← OpenVINO/RK3588 only
}
location /ai-inference/ {
    proxy_pass http://ai_inference:8001/;     # ← OpenVINO/RK3588 only
}
```

nginx resolves all upstream hostnames at startup. On Jetson, `video_pipeline` and `ai_inference` containers don't exist — only `deepstream_service:8001` runs. nginx fails immediately with:

```
[emerg] host not found in upstream "video_pipeline" in /etc/nginx/conf.d/default.conf:74
```

### Fix applied

1. Created `nginx/conf.d.jetson/default.conf` — identical to the shared config except both `/video-pipeline/` and `/ai-inference/` proxy to `deepstream_service:8001`
2. Updated `docker-compose.jetson.yml` nginx volume mount:
   ```diff
   - ./nginx/conf.d:/etc/nginx/conf.d:ro
   + ./nginx/conf.d.jetson:/etc/nginx/conf.d:ro
   ```

The shared `nginx/conf.d/default.conf` (used by OpenVINO and RK3588) is unchanged.

---

## 7. What Was NOT Tested (Requires Hardware/Models)

| Item | Reason | How to test |
|---|---|---|
| TRT engine generation | PeopleNet + ReID ONNX not present | Download via `scripts/download_peoplenet.sh` + `download_reid.sh` (requires NGC_API_KEY) |
| Live person detection | No RTSP camera connected | Register a real RTSP URL, observe `Launched core_engine PID` + detections at `/shared/cameras/{id}/detections/latest` |
| ReID cross-camera matching | Requires two cameras + valid embeddings | Run `reid_probe.py` — expect coverage ≥100%, inter-ID cosine < 0.55 |
| Loitering / entrance-exit events | Requires person detections + configured zones | Configure zone in UI, walk person through zone |
| ZMQ data flow end-to-end | No real GStreamer pipeline | With models present, check `reid_probe.py` embedding quality stats |

---

## 8. Submodule Changes on This Branch

### `services/deepstream-jetson` (`fix/download-reid-create-models-dir`)

| File | Change |
|---|---|
| `core_engine/CMakeLists.txt` | `cmake_minimum_required` 3.16 → 3.13 (DeepStream L4T base has CMake 3.13) |
| `requirements.txt` | Added `python-multipart>=0.0.9` (required by FastAPI `Form(...)` endpoints) |
| `.dockerignore` | Added — excludes `core_engine/build/`, `models/`, `__pycache__/`, `.git/` |
| `tests/test_deepstream_api.py` | New — 39 unit tests for API layer |
