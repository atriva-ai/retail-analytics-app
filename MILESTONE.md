# 🎯 MILESTONE: Dashboard Analytics MVP

**Goal:** Populate all Dashboard KPIs and charts with real data from the DeepStream AI pipeline.

---

## FEATURE 1 — Entrance / Exit Counting (Foundation)

> Enables: **Total Visitors, Occupancy, Peak Hour, Traffic Chart**

### AI / Inference

* [x] Define entrance virtual line per camera (line-drawing canvas in Analytics Engines settings)
* [x] Implement centroid direction detection (DeepStream NvDCF tracker + centroid crossing logic)
* [x] Detect line crossing events (`line_cross_in`, `line_cross_out`)
* [x] Classify event as `IN` or `OUT`
* [x] Debounce duplicate crossings (per-track debounce in backend analytics engine)

### Backend / DB

* [x] `analytics_events` table — `(id, camera_id, analytics_id, event_type, value, created_at)`
* [x] Persist IN / OUT events via analytics engine pipeline

### API

* [x] `GET /api/v1/analytics-events/?event_type=line_cross_in&...` (with time range + camera filters)
* [x] `GET /api/v1/analytics-events/summary` — total_in, total_out, current_zone_count per engine
* [x] `GET /api/v1/analytics-events/count-by-hour` (count_by_hour CRUD function)

### Frontend / Dashboard UI

* [x] Display **Total Visitors** KPI (sum of `total_in` across entrance/line analytics)
* [x] Display **Current Occupancy** KPI (sum of `current_zone_count` across zone analytics)
* [x] Display **Peak Hour** KPI (derived from hourly bucketing of `line_cross_in` events)

✅ **Complete** — IN/OUT counters, occupancy, and peak hour all wired to real `analytics_events` data.

---

## FEATURE 2 — Visitors Over Time (Traffic Chart)

> Depends on: **Entrance / Exit Counting**

### Backend / Aggregation

* [x] Aggregate entry events hourly (bucketed by `created_at` hour in frontend)
* [x] Support Today / Week / Month range (time range passed as ISO query params)
* [ ] Server-side cache for hourly aggregation (currently computed client-side from raw events)

### API

* [x] `GET /api/v1/analytics-events/?event_type=line_cross_in&limit=500&start_time=...&end_time=...`

### Frontend / Dashboard UI

* [x] Bar chart: Visitors per hour (`DashboardChart`)
* [x] Today filter clips future hours (renders only hours ≤ current hour)
* [x] Loading skeleton and empty state ("Configure an Entrance analytics engine in Settings")

✅ **Complete** — chart driven by real events; server-side caching is a future perf optimisation only.

---

## FEATURE 3 — Tracking Lifecycle (Session Definition)

> Enables: **Dwell Time & Person Activity**

### AI / Inference

* [x] IoU + centroid tracker via DeepStream NvDCF (`config_tracker_NvDCF.yml`)
* [x] Ephemeral `track_id` per detection (from `NvDsObjectMeta.object_id`)
* [x] Track creation and expiration handled by NvDCF
* [x] Cross-camera Re-ID assigns `global_id` (ResNet50 ReID model)

### Backend / DB

* [x] Tracks implicitly stored as analytics events with `track_id` in `value` JSON
* [ ] Dedicated `tracks` table with `(track_id, camera_id, start_ts, end_ts)` — not yet created; dwell computed from event timestamps instead

### API

* [x] Track activity visible via `GET /api/v1/analytics-events/` (per-event records)
* [x] Person Activity page shows full event log with track IDs

### Frontend (Validation Only)

* [x] Track ID visible in Person Activity event table
* [x] Live detection count per camera via WebSocket (`/ws/cameras/{id}/live`)

✅ **Functionally complete** — dedicated `tracks` table deferred; dwell computed from `dwell_alert` event values.

---

## FEATURE 4 — Dwell Time (Average)

> Depends on: **Tracking Lifecycle**

### Backend / Calculation

* [x] Dwell time computed per track by the loitering/dwell analytics engine
* [x] Stored as `dwell_alert` event with `{"track_id": N, "dwell_s": 47.2}` in `value`
* [x] Short-lived tracks naturally excluded (dwell engine has minimum threshold config)

### API

* [x] `GET /api/v1/analytics-events/?event_type=dwell_alert&...` — returns dwell events with `dwell_s`

### Frontend / Dashboard UI

* [x] Display **Avg Dwell Time** KPI (average `dwell_s` across `dwell_alert` events in range)
* [x] Unit formatting: seconds → `Xm Ys` or `Xs`

✅ **Complete** — dwell KPI driven by real `dwell_alert` events from the analytics engine.

---

## FEATURE 5 — Zone Dwell & Engagement

> Enables: **Zone Engagement Chart**

### AI / Inference

* [x] Point-in-polygon zone detection (zone polygon drawn via `ZoneDrawingCanvas`)
* [x] Zone enter / exit events (`zone_enter`, `zone_exit`, `zone_count_snapshot`)
* [x] Dwell time per zone tracked by `dwell_time_by_zone` analytics engine

### Backend / DB

* [x] Zone events stored in `analytics_events` with zone coordinates in engine `config` JSON
* [x] `current_zone_count` tracked in summary endpoint

### Backend / Aggregation

* [x] Zone event count per engine via `GET /api/v1/analytics-events/summary`

### API

* [x] `GET /api/v1/analytics-events/summary` — `event_count`, `current_zone_count`, `last_event_at`

### Frontend / Dashboard UI

* [x] Bar chart: Analytics Activity by Engine (`ZoneEngagement` component)
* [x] Sorted by event count, top 8 engines shown
* [x] Updates with Today / Week / Month filter

✅ **Complete** — zone engagement chart driven by real summary data.

---

## FEATURE 6 — Peak Hour Calculation

> Depends on: **Entrance / Exit Counting**

### Backend / Logic

* [x] Entry events bucketed by hour client-side (`formatHour` on `created_at`)
* [x] Hour with max entry count identified from `line_cross_in` events
* [ ] Dedicated server-side `/kpi/peak-hour` endpoint — not built; computed client-side

### Frontend / Dashboard UI

* [x] Display **Peak Hour** KPI with entry count sub-label
* [x] Scoped to selected time filter (Today / Week / Month)

✅ **Complete** — peak hour computed correctly from real data; server endpoint is a future optimisation.

---

## FEATURE 7 — Camera Status Summary

> Independent of analytics

### Backend

* [x] Camera heartbeat via `decode-status` endpoint (polls DeepStream/video-pipeline)
* [x] Fields: `streaming_status`, `is_active`, `frame_count`, `last_error`
* [x] `streaming_status` values: `stopped`, `starting`, `streaming`, `stopping`, `error`

### API

* [x] `GET /api/v1/cameras/` — list all cameras
* [x] `GET /api/v1/cameras/{id}/decode-status/` — live streaming status + frame count
* [x] `POST /api/v1/cameras/{id}/activate/` and `deactivate/`

### Frontend / Dashboard UI & Settings

* [x] Camera Status Table on Dashboard (`CameraStatusTable`)
* [x] Camera grid on Live Cameras page with per-camera streaming status badge
* [x] Start/Stop Streaming buttons with polling (3 s intervals, 10 max polls) for status stabilisation
* [x] Snapshot preview dialog (latest JPEG from OSD MKV via GStreamer)

✅ **Complete** — camera status live from DeepStream decode pipeline.

---

## FEATURE 8 — Dashboard Time Range Control

> Cross-cutting feature

### Frontend

* [x] Today / Week / Month tabs on Dashboard header
* [x] Filter applied to: Total Visitors, Avg Dwell, Traffic Chart, Zone Engagement
* [x] Current Occupancy correctly excluded from time filter (always real-time)
* [x] Peak Hour scoped to selected range

### Backend

* [x] `start_time` / `end_time` ISO query params consistently supported across all analytics event endpoints
* [x] Summary endpoint accepts time range params

✅ **Complete** — time filter wired end-to-end.

---

## ADDITIONAL FEATURES (beyond original scope)

### Settings — Analytics Engines (Phase D)
* [x] Camera-scoped analytics engine panel (select camera → see / add / toggle / delete engines)
* [x] Line-drawing canvas for line-based analytics (entrance, line cross count)
* [x] Zone-drawing canvas for zone-based analytics (people counting by zone, dwell time by zone)
* [x] Entrance direction selector (In / Out / Both)
* [x] 8 analytics engine types supported

### Settings — Alert Engines
* [x] Alert types: Loitering, Overcrowding, Restricted Area, Queue Length, Human Detection, Line Crossing
* [x] Zone / line drawing for alert zone configuration
* [x] Toggle alert engines per camera

### Settings — Cameras
* [x] Add / Edit / Delete cameras
* [x] RTSP stream validation on add
* [x] AI person detection toggle per camera
* [x] Snapshot frame preview modal

### Live Cameras Page
* [x] Camera grid with live streaming status
* [x] Per-camera WebSocket — live person detection count badge
* [x] Exponential back-off reconnect (1 s → 30 s max)

### Person Activity Page
* [x] Event log table with pagination
* [x] Filter by camera and analytics engine
* [x] Sortable columns (created_at)
* [x] Event type labels (Enter, Exit, Zone Enter, Zone Exit, Dwell Alert, Headcount)
* [x] Entrance / Exit cumulative count summary component

### WebSocket Live Detection (Phase C)
* [x] Backend WebSocket endpoint `/ws/cameras/{id}/live`
* [x] Pushes `{type: "detections", camera_id, count, ts}` on every inference frame
* [x] Frontend `useWebSocket` hook with exponential back-off reconnect

### Backend — Startup Recovery
* [x] On backend reboot, AI inference automatically restarted for all active person-detection cameras

---

# 🧩 GitHub Milestone Mapping

| Milestone | Features | Status |
|-----------|----------|--------|
| 1 | Feature 1, 2 | ✅ Complete |
| 2 | Feature 3, 4 | ✅ Complete |
| 3 | Feature 5 | ✅ Complete |
| 4 | Feature 6, 7, 8 | ✅ Complete |
| 5 (stretch) | Alert Engines, Person Activity, Live Cameras WebSocket | ✅ Complete |

---

# 🔑 Outstanding / Next

| Item | Priority | Notes |
|------|----------|-------|
| Server-side hourly aggregation cache | Low | Dashboard loads up to 500 events client-side; fine for MVP |
| Dedicated `tracks` table | Low | Dwell computed from events; tracks table adds query power |
| `count_by_hour` API route | Medium | CRUD function done; route + frontend chart integration pending |
| Staff Monitoring page | Medium | Page stub exists (`/staff-monitoring`) |
| Product Analytics page | Medium | Page stub exists (`/product-analytics`) |
| Incomplete `count_by_hour` body | Fixed | Implemented in `backend/app/db/crud/analytics_event.py` |
