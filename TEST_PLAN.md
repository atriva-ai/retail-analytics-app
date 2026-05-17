# UI Test Plan — Retail Analytics MVP

**Scope:** All features accessible from the browser UI.
**Prerequisites:** Stack running via `docker compose -f docker-compose.jetson.yml up`, at least one RTSP camera available, DeepStream pipeline healthy.

---

## 0 — Prerequisites / Smoke

| # | Step | Expected |
|---|------|----------|
| 0.1 | Open `http://localhost` | App loads; sidebar shows Dashboard, Live Cameras, Person Activity, Settings |
| 0.2 | Open browser DevTools → Network tab | No 5xx errors on initial page load |
| 0.3 | Check backend health: `curl http://localhost/api/v1/cameras/` | Returns JSON array (empty OK) |

---

## 1 — Settings › Cameras

### 1.1 Add Camera

| # | Step | Expected |
|---|------|----------|
| 1.1.1 | Settings → Cameras tab → click **Add Camera** | Dialog opens with Name, RTSP URL, Location, Activate toggle |
| 1.1.2 | Fill Name + valid RTSP URL, click **Add Camera** | Toast "Camera added successfully"; camera appears in table |
| 1.1.3 | Open Add dialog, enter an unreachable RTSP URL, submit | Toast "Video Stream Issues" with error detail; camera still added |
| 1.1.4 | Submit with empty Name | Button should not submit (field required) |

### 1.2 Edit Camera

| # | Step | Expected |
|---|------|----------|
| 1.2.1 | Click edit (pencil) icon on any camera | Edit dialog opens pre-filled with current values |
| 1.2.2 | Change Name, click **Save Changes** | Toast "Camera updated successfully"; table shows new name |
| 1.2.3 | Toggle **AI Detection** switch in edit dialog, save | Badge in AI Detection column updates accordingly |

### 1.3 Streaming Controls

| # | Step | Expected |
|---|------|----------|
| 1.3.1 | Click **Start Streaming** on a stopped camera | Button changes to disabled "Starting…"; status shows "Starting..." in yellow |
| 1.3.2 | Wait ~5–10 s after starting | Status transitions to "Streaming" in green; frame count appears |
| 1.3.3 | Click **Refresh** while streaming | Toast "Status Updated"; frame count increments |
| 1.3.4 | Click **Stop Streaming** on a streaming camera | Status transitions to "Stopping…" then "Stopped" |
| 1.3.5 | Start → Stop → Start on same camera | No error; camera reaches Streaming state each time |
| 1.3.6 | Click **Start Streaming** on unreachable camera | Status shows "Error" in red within ~30 s |

### 1.4 Snapshot Preview

| # | Step | Expected |
|---|------|----------|
| 1.4.1 | Camera icon (snapshot) disabled when frame count = 0 | Icon grayed out, not clickable |
| 1.4.2 | After camera is streaming (frame count > 0), click snapshot icon | Preview dialog opens; spinner while loading |
| 1.4.3 | Snapshot loads | JPEG image displayed; frame count shown above image |
| 1.4.4 | Click **Close** | Dialog closes; no memory leak (object URL revoked) |

### 1.5 Delete Camera

| # | Step | Expected |
|---|------|----------|
| 1.5.1 | Click trash icon on a camera | Camera removed from table; toast "Camera deleted successfully" |
| 1.5.2 | Delete a camera that has analytics engines configured | Camera removed; analytics engines for that camera also gone (verify in Analytics tab) |

### 1.6 AI Detection Toggle (inline)

| # | Step | Expected |
|---|------|----------|
| 1.6.1 | Toggle AI Detection switch directly in table (without opening edit dialog) | Badge flips Enabled/Disabled; toast "AI Detection enabled/disabled" |

---

## 2 — Settings › Analytics Engines

### 2.1 Camera Selection

| # | Step | Expected |
|---|------|----------|
| 2.1.1 | Settings → Analytics tab with no cameras | "No cameras configured" message in left panel |
| 2.1.2 | With cameras present, left panel shows camera list | First camera auto-selected; right panel shows its configured analytics |

### 2.2 Add — Line-based Analytics (Entrance / Line Cross Count)

| # | Step | Expected |
|---|------|----------|
| 2.2.1 | Select camera → click **Entrance** type in Add grid | Name auto-fills; direction buttons (Enter/Exit/Both) appear; "Draw line on camera" button appears |
| 2.2.2 | Click **Save Analytics** without drawing line | Toast "Draw a line on the camera first" (validation error) |
| 2.2.3 | Click **Draw line on camera** | Canvas loads with a live camera snapshot as background |
| 2.2.4 | Click-drag to draw a line across the frame | Line rendered; "Line set: (x1,y1)→(x2,y2)" confirmation row appears with Redraw option |
| 2.2.5 | For Entrance type, verify entrance side prompt | After drawing line, secondary click to mark entrance side |
| 2.2.6 | Select direction (In / Out / Both), enter name, click **Save Analytics** | Engine appears in the list for that camera; toast "Saved" |
| 2.2.7 | Click **Redraw** on a set line | Canvas re-opens; new line replaces old |

### 2.3 Add — Zone-based Analytics (Dwell Time by Zone / People Counting by Zone)

| # | Step | Expected |
|---|------|----------|
| 2.3.1 | Click **People Counting by Zone** | "Draw zone on camera" button appears; no direction selector |
| 2.3.2 | Click **Save Analytics** without drawing zone | Toast "Draw a zone on the camera first" |
| 2.3.3 | Click **Draw zone on camera** | Zone canvas loads with camera snapshot background |
| 2.3.4 | Draw a polygon zone | Zone highlighted; confirmation row shows type + point count |
| 2.3.5 | Enter name, save | Engine appears in camera analytics list |

### 2.4 Toggle / Delete Analytics Engine

| # | Step | Expected |
|---|------|----------|
| 2.4.1 | Toggle Switch on an active engine | Switch turns off; engine `is_active` = false |
| 2.4.2 | Toggle back on | Switch turns on |
| 2.4.3 | Click trash icon on engine | Engine removed from list; toast "Analytics engine deleted" |

### 2.5 Multiple Analytics on Same Camera

| # | Step | Expected |
|---|------|----------|
| 2.5.1 | Add Entrance + Dwell Time by Zone to same camera | Both engines listed; count shows "2 active · 2 total" in left panel |
| 2.5.2 | Disable one engine | Count shows "1 active · 2 total" |

---

## 3 — Settings › Alert Engines

| # | Step | Expected |
|---|------|----------|
| 3.1 | Settings → Alert Engines tab | Camera list + alert type grid similar to Analytics |
| 3.2 | Select **Loitering** alert type | Requires zone; "Draw zone on camera" appears |
| 3.3 | Draw zone + set dwell threshold + save | Alert engine appears in list |
| 3.4 | Select **Overcrowding** | Requires zone + max count config |
| 3.5 | Select **Restricted Area** | Requires zone; no threshold |
| 3.6 | Toggle alert engine active/inactive | Switch updates; UI reflects change |
| 3.7 | Delete alert engine | Removed from list; toast confirmation |

---

## 4 — Dashboard

### 4.1 KPI Cards (requires active analytics)

| # | Step | Expected |
|---|------|----------|
| 4.1.1 | Load Dashboard with no analytics data | All 4 KPI cards show "—" with no errors |
| 4.1.2 | After entrance analytics generates events: **Total Visitors** | Shows cumulative `line_cross_in` count for selected range |
| 4.1.3 | After zone analytics generates events: **Current Occupancy** | Shows sum of `current_zone_count` from zone engines |
| 4.1.4 | After dwell alerts: **Avg Dwell Time** | Shows `Xm Ys` or `Xs` format averaged from `dwell_s` values |
| 4.1.5 | After sufficient entry events: **Peak Hour** | Shows AM/PM label (e.g. "2PM") with "N entries" sub-label |

### 4.2 Time Filter

| # | Step | Expected |
|---|------|----------|
| 4.2.1 | Click **Today** tab | KPIs and charts scoped to midnight → now |
| 4.2.2 | Click **Week** tab | Scoped to 7 days ago → now; Total Visitors increases |
| 4.2.3 | Click **Month** tab | Scoped to 30 days ago → now |
| 4.2.4 | Switch from Month → Today | Current Occupancy stays same (real-time); other KPIs reduce |

### 4.3 Charts

| # | Step | Expected |
|---|------|----------|
| 4.3.1 | With no entry data: Traffic chart | Empty state: "No entry events yet — configure an Entrance analytics engine" |
| 4.3.2 | With entry data: **Visitors Over Time** chart | Bar chart with AM/PM hour labels; bars reflect hourly entry counts |
| 4.3.3 | Today filter: bars only up to current hour | Future hours not shown |
| 4.3.4 | With no analytics events: **Analytics Activity** chart | Empty state message |
| 4.3.5 | With events: **Analytics Activity by Engine** chart | Horizontal bar chart; engines sorted by event count descending |

### 4.4 Camera Status Table

| # | Step | Expected |
|---|------|----------|
| 4.4.1 | No cameras | Table empty or "No cameras" message |
| 4.4.2 | Camera streaming | Row shows Online badge, FPS, Last Seen timestamp |
| 4.4.3 | Camera stopped | Row shows Offline or Stopped badge |

### 4.5 Recent Alerts

| # | Step | Expected |
|---|------|----------|
| 4.5.1 | No dwell alerts generated | "No alerts triggered yet" message with setup instruction |
| 4.5.2 | After loitering/dwell alert fires | Alert row shows: amber icon, type label, analytics engine badge, camera name, dwell duration, time |
| 4.5.3 | Up to 10 alerts shown | Most recent 10 only |

### 4.6 Manual Refresh

| # | Step | Expected |
|---|------|----------|
| 4.6.1 | Click refresh icon (top right) | Icon animates (spin); data reloads; new data reflected |
| 4.6.2 | Click refresh again immediately | Debounced OK (no double-fetch error) |

---

## 5 — Live Cameras

### 5.1 Camera Grid

| # | Step | Expected |
|---|------|----------|
| 5.1.1 | Navigate to Live Cameras | Grid shows one card per registered camera |
| 5.1.2 | Camera with `streaming_status = stopped` | Badge shows "offline" in red |
| 5.1.3 | Camera streaming | Badge shows "online" in green |

### 5.2 WebSocket Live Detection Count

| # | Step | Expected |
|---|------|----------|
| 5.2.1 | Camera with AI Detection enabled, streaming | Green pulsing dot appears next to card title |
| 5.2.2 | Person visible to camera | Live count badge updates (may take 1–2 s) |
| 5.2.3 | AI Detection disabled on camera | Dot grayed out; count not shown (WS not opened) |
| 5.2.4 | Stop camera stream | Dot grays out; WS reconnects with back-off (DevTools → WS tab shows reconnect attempts) |
| 5.2.5 | Network briefly disconnected then restored | WS reconnects automatically within 30 s; count resumes |

---

## 6 — Person Activity

### 6.1 Event Table

| # | Step | Expected |
|---|------|----------|
| 6.1.1 | Navigate to Person Activity with no events | Empty table or "No events" state |
| 6.1.2 | With events, default view | Table shows all event types; columns: Time, Camera, Analytics, Type, Value |
| 6.1.3 | Filter by camera dropdown | Only events for that camera shown |
| 6.1.4 | Event type displayed correctly | `line_cross_in` → "Enter", `dwell_alert` → "Dwell Alert", etc. |

### 6.2 Pagination

| # | Step | Expected |
|---|------|----------|
| 6.2.1 | More than one page of events | Pagination controls visible |
| 6.2.2 | Click Next page | Next page of events loads; page number increments |
| 6.2.3 | Click Previous | Returns to previous page |

### 6.3 Sort

| # | Step | Expected |
|---|------|----------|
| 6.3.1 | Click column header (e.g. Time) | Events re-sorted ascending |
| 6.3.2 | Click again | Re-sorted descending |

### 6.4 Entrance / Exit Summary

| # | Step | Expected |
|---|------|----------|
| 6.4.1 | With entrance analytics events | Summary row shows total IN / total OUT counts |
| 6.4.2 | Filter by camera with entrance analytics | Counts update for that camera only |

---

## 7 — Cross-feature / Regression

| # | Step | Expected |
|---|------|----------|
| 7.1 | Add camera → Start streaming → add Entrance analytics → trigger crossings → check Dashboard | Dashboard KPIs reflect real events; no stale "—" states |
| 7.2 | Delete analytics engine → reload Dashboard | Dashboard no longer counts events for deleted engine |
| 7.3 | Stop camera stream → reload Person Activity | No new events added for stopped camera |
| 7.4 | Add 3+ cameras | Camera grid wraps correctly; analytics panel camera list scrolls; Dashboard camera table shows all |
| 7.5 | Rapidly toggle analytics engine on/off 5× | No console errors; final state consistent with last toggle |
| 7.6 | Refresh browser on Settings page | Page reloads to correct tab; data re-fetched without error |
| 7.7 | Open Dashboard in two browser tabs | Both update independently on manual refresh; no shared state corruption |

---

## 8 — Empty / Error States

| # | Step | Expected |
|---|------|----------|
| 8.1 | Load app with backend down | Error toasts visible; pages show loading or graceful empty state (no white screen crash) |
| 8.2 | Start streaming with no DeepStream running | Camera status transitions to "Error"; Dashboard camera table shows offline |
| 8.3 | Snapshot with pipeline not running | Snapshot icon disabled (frame count = 0); if forced, 503/404 handled gracefully |
| 8.4 | Add duplicate camera name | Camera added (names not unique-constrained); both appear in list |

---

## Test Environment Notes

- Run stack: `docker compose -f docker-compose.jetson.yml up`
- Seed test data (no live cameras): Use `POST /api/v1/analytics-events/` to insert synthetic events directly, or connect an RTSP test stream (e.g. `rtsp://wowzaec2demo.streamlock.net/vod/mp4:BigBuckBunny_115k.mov`)
- WebSocket tests require a streaming camera with AI detection enabled
- Snapshot tests require DeepStream pipeline running and OSD MKV being written to `/tmp/reid_annotated.mkv`
