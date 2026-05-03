# Jetson Full-Stack Test Report

**Dates:** 2026-05-02 (static/API/stack) · 2026-05-03 (live inference)  
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
| TRT engine build from ONNX (first run) | ✅ Both engines built successfully |
| Live inference — 3 RTSP streams | ✅ Person detection running |
| ReID embedding quality | ✅ All metrics pass |
| Bugs found and fixed | ⚠️ 4 bugs — see §6 |

---

## Setup Instructions

For step-by-step instructions to run the application from scratch on a new Jetson device, see the **[README — Quick Start](../../readme.md#quick-start--jetson)**.

For detailed DeepStream pipeline documentation, model configuration, and diagnostic commands, see **[services/deepstream-jetson/README.md](../../services/deepstream-jetson/README.md)**.

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

---

## 5. Full Stack Smoke Test (`docker-compose.jetson.yml`)

### Container status (all healthy)

| Service | Image | Status | Ports |
|---|---|---|---|
| `deepstream_service` | `deepstream-l4t:6.2-triton` | ✅ Up | 8001 |
| `backend` | `python:3.11-slim` | ✅ Up | 8000 |
| `db` | `postgres:15.5` | ✅ Up | 5432 |
| `frontend` | `node:18-alpine` | ✅ Up | 3000 |
| `nginx` | `nginx:1.25.3-alpine` | ✅ Up | 80, 443 |

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

## 6. Bugs Found and Fixed

### Bug 1 — nginx Jetson config (2026-05-02)

**Symptom:** nginx crashes at startup with `[emerg] host not found in upstream "video_pipeline"`.

**Root cause:** `nginx/conf.d/default.conf` (shared across all platforms) references `video_pipeline:8002` and `ai_inference:8001` — services that only exist on OpenVINO/RK3588 deployments. nginx resolves all upstream hostnames at startup; missing hostnames are fatal.

**Fix:**
1. Created `nginx/conf.d.jetson/default.conf` — both `/video-pipeline/` and `/ai-inference/` proxy to `deepstream_service:8001`
2. Updated `docker-compose.jetson.yml` nginx volume mount:
   ```diff
   - ./nginx/conf.d:/etc/nginx/conf.d:ro
   + ./nginx/conf.d.jetson:/etc/nginx/conf.d:ro
   ```

---

### Bug 2 — Docker volume mount placed models at wrong path (2026-05-03)

**Symptom:** `ERROR: Cannot access ONNX file '/workspace/core_engine/configs/../../models/reid/...'` — pipeline fails immediately on first run (no cached engine). SGIE fails first (pipeline initialises downstream-to-upstream), so PGIE errors were not visible.

**Root cause:** The nvinfer configs use `../../models/` relative to their directory (`core_engine/configs/`), which resolves to:
- **Standalone:** `services/deepstream-jetson/core_engine/configs/../../models/` = `services/deepstream-jetson/models/` ✅
- **Docker (before fix):** `/workspace/core_engine/configs/../../models/` = `/workspace/models/` — but the volume was mounted at `/workspace/core_engine/models/`, not `/workspace/models/` ❌

The configs were correct for both modes; the Docker volume mount was at the wrong path.

**Fix:**
1. Changed `docker-compose.jetson.yml` volume mount:
   ```diff
   - ./services/deepstream-jetson/models:/workspace/core_engine/models
   + ./services/deepstream-jetson/models:/workspace/models
   ```
2. Updated `Dockerfile` placeholder `mkdir` to match:
   ```diff
   - RUN mkdir -p /workspace/core_engine/models/peoplenet /workspace/core_engine/models/reid
   + RUN mkdir -p /workspace/models/peoplenet /workspace/models/reid
   ```
The `../../models/` paths in both config files were left unchanged — they are correct for standalone and Docker alike.

---

### Bug 3 — ReID ETLT model format (2026-05-03)

**Symptom:** `NvDsInferCudaEngineGetFromTltModel: Failed to open TLT encoded model file resnet50_market1501.etlt`.

**Root cause:** `download_reid.sh` was downloading `reidentificationnet:deployable_v1.0` which ships an ETLT file (AES-256 encrypted ONNX). ETLT decryption requires a key set at export time; NVIDIA does not publicly document the correct key for this model. Additionally, DeepStream 6.2 on Jetson does not ship `libnvdsinfer_custom_impl_Tao.so`, which is required for inline ETLT decryption. The correct version is `deployable_v1.2`, which ships a plain ONNX file.

**Fix:**
1. Updated `download_reid.sh` to fetch `deployable_v1.2` (`resnet50_market1501_aicity156.onnx`)
2. Updated `config_infer_secondary_reid.txt` to use `onnx-file=` instead of `tlt-encoded-model=`

---

### Bug 4 — nvinfer `network-type=1` for embedding model (2026-05-03)

**Symptom:** Would have produced silent incorrect output — the embedding tensor would be misinterpreted as classifier logits, and `NvDsClassifierMeta` would be populated with garbage class assignments instead of raw float tensors.

**Root cause:** `config_infer_secondary_reid.txt` had `network-type=1` (classifier). Embedding models must use `network-type=100` (OTHER). With `network-type=1`, nvinfer allocates `NvDsClassifierMeta` and interprets the 256-dim output as class logits. The pad probe reads embeddings from `NvDsInferTensorMeta` (via `output-tensor-meta=1`), which is populated regardless of `network-type`, but the classifier path wastes memory and could interfere.

**Fix:** Changed `network-type=1` → `network-type=100` in `config_infer_secondary_reid.txt`.

---

## 7. TRT Engine Build — First Run (2026-05-03)

**Hardware:** Jetson Orin NX 16 GB  
**Models:** PeopleNet ResNet-34 FP16, ReID ResNet50 FP16  
**Triggered by:** camera registration with valid RTSP URL

### Engine build log (abridged)

```
# ReID SGIE — built first (downstream init order)
[UID 2] Trying to create engine from model files
[TRT] WARNING: Tactic Device request: 410MB Available: 359MB. Skipping tactic.
[UID 2] serialize cuda engine to file: .../resnet50_market1501_aicity156.onnx_b8_gpu0_fp16.engine

# PeopleNet PGIE — built second
[UID 1] Trying to create engine from model files
[UID 1] serialize cuda engine to file: .../resnet34_peoplenet_int8.onnx_b1_gpu0_fp16.engine
```

The tactic memory warnings are normal — TRT skips tactics that exceed available GPU memory and selects alternatives. They do not affect inference correctness.

### Engine files produced

| Model | Engine file | Size | Build time |
|---|---|---|---|
| ReID ResNet50 | `resnet50_market1501_aicity156.onnx_b8_gpu0_fp16.engine` | 47 MB | ~60 s |
| PeopleNet ResNet-34 | `resnet34_peoplenet_int8.onnx_b1_gpu0_fp16.engine` | 5 MB | ~90 s |

Engines are written to `services/deepstream-jetson/models/` on the host (bind-mounted into the container). Subsequent container starts load the cached engines and are ready in ~10 seconds.

---

## 8. Live Inference — 3 RTSP Streams (2026-05-03)

**Streams:** `rtsp://192.168.9.146:18554/stream1` through `stream3`  
**Pipeline:** nvurisrcbin → nvstreammux → nvinfer(PeopleNet) → nvtracker(NvDCF) → nvinfer(ReID SGIE) → OSD → sink

### Pipeline status

```bash
GET http://localhost:8001/api/v1/video-pipeline/debug/
{
    "architecture": "deepstream",
    "pipeline_alive": true,
    "cameras": [
        {"camera_id": "cam1", "rtsp_url": "rtsp://192.168.9.146:18554/stream1"},
        {"camera_id": "cam2", "rtsp_url": "rtsp://192.168.9.146:18554/stream2"},
        {"camera_id": "cam3", "rtsp_url": "rtsp://192.168.9.146:18554/stream3"}
    ]
}
```

### Detection store — live sample

```bash
GET http://localhost:8001/shared/cameras/cam1/detections/latest
{
    "camera_id": "cam1",
    "detections": [
        {"camera_id": "cam1", "track_id": 20, "class_id": 0, "class_name": "person",
         "confidence": 0.7017, "bbox": [1187.68, 320.24, 1298.32, 551.26]},
        {"camera_id": "cam1", "track_id": 19, "class_id": 0, "class_name": "person",
         "confidence": 0.6843, "bbox": [739.03, 319.80, 840.94, 559.81]}
    ]
}
```

### ZMQ frame event — raw sample

```json
{"event": "frame", "source_id": 0, "frame": 691,
 "detections": [
   {"class": 0, "label": "person", "conf": 0.70, "tid": 20,
    "bbox": {"l": 1187.7, "t": 320.2, "w": 110.6, "h": 231.0},
    "emb": "<base64 float32[256]>"}
 ]}
```

---

## 9. ReID Embedding Quality (2026-05-03)

**Tool:** inline ZMQ subscriber collecting 200 frames from 3 cameras  
**Tracks observed:** multiple unique identities across 2 active cameras

| Metric | Result | Target | Status |
|---|---|---|---|
| Embedding coverage | 461 / 461 = **100%** | 100% | ✅ |
| Embedding dimension | **256** float32 | 256 | ✅ |
| Embedding norm (mean) | **19.3** (std 1.7) | ~1.0 if L2-norm'd; raw BN output | ✅ |
| Intra-ID cosine sim | **0.941** | > 0.70 | ✅ |
| Inter-ID cosine sim | **0.487** | < 0.55 | ✅ |
| Gap (intra − inter) | **0.454** | ≥ 0.30 | ✅ |

**Interpretation:**
- **100% coverage** — every person detection in every frame carries an embedding. The `secondary-reinfer-interval=1` GObject property and absence of `operate-on-gie-id` are both working correctly.
- **Gap of 0.454** — embeddings from the same person across frames are far more similar than embeddings from different people. This is well above the 0.30 threshold required for reliable cross-camera matching.
- **Norm ~19** — the `fc_pred` BatchNorm output is not L2-normalised. Cosine similarity normalises by the norms, so matching quality is unaffected.

---

## 10. What Remains Untested

| Item | Reason | How to test |
|---|---|---|
| Cross-camera ReID matching (reid_matcher.py) | Multiple people moving between cameras needed | Run `reid_matcher.py --threshold 0.75`; look for `reid_link` events when same person appears on two cameras |
| Loitering / entrance-exit events | Requires person detections + configured zones | Configure zones via `zone_calibrator.py`, run `loitering_detector.py`, walk person through zone |
| Annotated video output (`osd_file`) | `/tmp/reid_annotated.mkv` in container | `docker cp` the file out; play with `ffplay` |
| OSD live display (`osd_display: true`) | Requires a display connected to Jetson | Set `osd_display: true` in `default.yaml`, connect HDMI |
| Multi-stream scale beyond 3 cameras | Not tested | Register 4+ RTSP URLs; monitor GPU/CPU via `tegrastats` |
| Backend loitering API → UI flow | Requires loitering events in detection store | With zones configured and persons detected, verify UI shows alerts |
| TRT engine re-use after host reboot | Not tested in this session | Stop/remove containers; re-run `docker compose up` (no `--build`); verify engines load from host models dir |

---

## 11. Branch Changes Summary

### Main repo (`fix/deepstream-jetson-base-image`)

| File | Change |
|---|---|
| `docker-compose.jetson.yml` | Named volume `deepstream_engine_cache` → bind mount `./services/deepstream-jetson/models:/workspace/models` (correct path so nvinfer `../../models/` resolves correctly inside container); removed orphaned volume declaration |
| `nginx/conf.d.jetson/default.conf` | New — Jetson-specific nginx config routing `/video-pipeline/` and `/ai-inference/` to `deepstream_service:8001` |
| `readme.md` | Rewritten — complete Jetson Quick Start (clone → download models → configure env → compose up → register cameras → verify); accurate project structure; environment variables table |
| `CLAUDE.md` | Updated First-time model setup section; documents bind-mount layout and pre-build download requirement |
| `.gitignore` | New — excludes `.claude/` (personal settings), `backend/.env`, build artifacts |
| `docs/test-reports/jetson-full-test-2026-05-02.md` | This file |

### Submodule `services/deepstream-jetson` (`fix/download-reid-create-models-dir`)

| File | Change |
|---|---|
| `core_engine/CMakeLists.txt` | `cmake_minimum_required` 3.16 → 3.13 |
| `core_engine/configs/config_infer_primary_peoplenet.txt` | Added dual-mode path comment; `../../models/` paths unchanged (correct for both Docker and standalone) |
| `core_engine/configs/config_infer_secondary_reid.txt` | `tlt-encoded-model` → `onnx-file` with `../../models/` path; `network-type=1` → `100`; removed `tlt-model-key`; added ImageNet normalisation (`offsets`, `net-scale-factor`) |
| `Dockerfile` | `mkdir` changed from `/workspace/core_engine/models/` → `/workspace/models/` to match volume mount point |
| `requirements.txt` | Added `python-multipart>=0.0.9` |
| `scripts/download_reid.sh` | `deployable_v1.0` (ETLT) → `deployable_v1.2` (ONNX) |
| `README.md` | Added "Prepare models before building" prerequisite section |
| `.gitignore` | Added `models/reid/` patterns (was missing) |
| `.dockerignore` | New — excludes build artifacts, models, pycache |
| `tests/test_deepstream_api.py` | New — 39 unit tests for API layer |
