# 🎯 MILESTONE: Dashboard Analytics MVP

**Goal:**
Populate all Dashboard KPIs and charts with real data:

* Total Visitors
* Visitors Over Time
* Current Occupancy
* Peak Hour
* Average Dwell Time
* Camera Status

---

## FEATURE 1 — Entrance / Exit Counting (Foundation)

> Enables: **Total Visitors, Occupancy, Peak Hour, Traffic Chart**

### AI / Inference

* [ ] Define entrance virtual line per camera
* [ ] Implement centroid direction detection
* [ ] Detect line crossing events
* [ ] Classify event as `IN` or `OUT`
* [ ] Debounce duplicate crossings

### Backend / DB

* [ ] Create `entry_exit_events` table

```
(id, camera_id, direction, timestamp)
```

* [ ] Persist IN / OUT events

### API

* [ ] `GET /events/entry-exit?camera_id=&range=`
* [ ] `GET /kpi/total-visitors?range=`
* [ ] `GET /kpi/occupancy`
* [ ] `GET /kpi/peak-hour`

### Frontend / Dashboard UI

* [ ] Display **Total Visitors** KPI
* [ ] Display **Current Occupancy** KPI
* [ ] Display **Peak Hour (Today)** KPI
* [ ] Add tooltip explaining each KPI

✅ **Completion Criteria**

* IN/OUT counters increment correctly
* Occupancy updates in real time
* Peak hour updates after traffic changes

---

## FEATURE 2 — Visitors Over Time (Traffic Chart)

> Depends on: **Entrance / Exit Counting**

### Backend / Aggregation

* [ ] Aggregate entry events hourly
* [ ] Support Today / Week / Month range
* [ ] Cache hourly results

### API

* [ ] `GET /charts/visitors-over-time?range=`

### Frontend / Dashboard UI

* [ ] Line chart: Visitors per hour
* [ ] Apply time range selector
* [ ] Show loading and empty states

✅ **Completion Criteria**

* Chart updates correctly when time range changes
* Hourly bins match backend aggregation

---

## FEATURE 3 — Tracking Lifecycle (Session Definition)

> Enables: **Dwell Time & Person Activity**

### AI / Inference

* [ ] Implement IoU + centroid tracker
* [ ] Assign ephemeral `track_id`
* [ ] Handle track creation and expiration
* [ ] Timestamp track start and end

### Backend / DB

* [ ] Create `tracks` table

```
(track_id, camera_id, start_ts, end_ts)
```

### API

* [ ] `GET /tracks?camera_id=&range=`

### Frontend (Validation Only)

* [ ] Display track ID on live camera view

✅ **Completion Criteria**

* Tracks persist correctly while visible
* End time recorded when track disappears

---

## FEATURE 4 — Dwell Time (Average)

> Depends on: **Tracking Lifecycle**

### Backend / Calculation

* [ ] Compute dwell time per track

```
dwell = end_ts - start_ts
```

* [ ] Exclude tracks shorter than threshold (noise)
* [ ] Aggregate average dwell per time range

### API

* [ ] `GET /kpi/avg-dwell?range=`

### Frontend / Dashboard UI

* [ ] Display **Average Dwell Time** KPI
* [ ] Add unit formatting (seconds → mm:ss)

✅ **Completion Criteria**

* Dwell values are stable
* Short-lived detections excluded

---

## FEATURE 5 — Zone Dwell & Engagement

> Enables: **Zone Engagement Chart**

### AI / Inference

* [ ] Implement point-in-polygon zone detection
* [ ] Detect zone enter / exit per track
* [ ] Track time spent per zone

### Backend / DB

* [ ] Create `zone_events` table

```
(track_id, zone_id, enter_ts, exit_ts)
```

### Backend / Aggregation

* [ ] Aggregate dwell per zone
* [ ] Calculate:

  * zone visits
  * avg dwell per zone

### API

* [ ] `GET /kpi/zones?range=`

### Frontend / Dashboard UI

* [ ] Bar chart: Zone engagement
* [ ] Tooltip with visit count + avg dwell

✅ **Completion Criteria**

* Zone stats reflect real movement
* Chart updates with time filter

---

## FEATURE 6 — Peak Hour Calculation

> Depends on: **Entrance / Exit Counting**

### Backend / Logic

* [ ] Group entry events by hour
* [ ] Identify hour with max entries
* [ ] Restrict calculation to Today

### API

* [ ] `GET /kpi/peak-hour`

### Frontend / Dashboard UI

* [ ] Display Peak Hour KPI
* [ ] Label as “Today”

✅ **Completion Criteria**

* Peak hour changes correctly throughout day

---

## FEATURE 7 — Camera Status Summary

> Independent of analytics

### Backend

* [ ] Track camera heartbeat
* [ ] Record:

  * last frame timestamp
  * FPS
  * analytics enabled flag

### API

* [ ] `GET /cameras/status`

### Frontend / Dashboard UI

* [ ] Camera status table:

  * Name
  * Online / Offline
  * FPS
  * Last Seen

✅ **Completion Criteria**

* Offline cameras detected
* Status updates without page refresh

---

## FEATURE 8 — Dashboard Time Range Control

> Cross-cutting feature

### Frontend

* [ ] Add Today / Week / Month selector
* [ ] Apply filter to:

  * Total Visitors
  * Avg Dwell
  * Visitors Over Time
  * Zone Engagement
* [ ] Exclude:

  * Current Occupancy
  * Peak Hour

### Backend

* [ ] Support range query params consistently

✅ **Completion Criteria**

* Filter behavior is consistent and documented

---

# 🧩 Suggested GitHub Milestones

### Milestone 1

* Feature 1, 2

### Milestone 2

* Feature 3, 4

### Milestone 3

* Feature 5

### Milestone 4

* Feature 6, 7, 8

---

# 🔑 Final Note (Important)

Each feature:

* Is **independently shippable**
* Unlocks visible dashboard progress
* Avoids ReID or complex ML
* Keeps MVP honest and defensible

---

If you want, next I can:

* Convert this into **GitHub Issue templates**
* Add **acceptance criteria per issue**
* Map features to **PR structure**
* Define **Pro features roadmap**

Just tell me what you want next.
