[README (1).md](https://github.com/user-attachments/files/29542197/README.1.md)
# Glow Monitor — real-time monitoring & reminder web app

A live dashboard that authenticates users, streams continuously-updating
"level" data (UV index, air quality, tank level — extensible) over
WebSockets, plots it on a live Google Map, and fires smart reminders based
on both the data and the clock.

---

## 1. System architecture

```
┌─────────────────────────┐        HTTPS (REST)        ┌──────────────────────────┐
│   Browser (client)      │ ─────────────────────────▶ │  Express API             │
│  index.html              │  /api/auth/signup, /login  │  routes/auth.js          │
│  - Login/Signup forms    │  /api/live/current          │  routes/live.js          │
│  - Personalized greeting │ ◀───────────────────────── │  middleware/auth.js (JWT)│
│  - Live metric cards     │                             └──────────┬───────────────┘
│  - Google Maps markers   │                                        │
│  - Reminder toasts       │        WebSocket (Socket.io)           │ Mongoose
│                          │ ◀═══════════════════════════════════▶  ▼
└──────────────────────────┘   'liveData' event, every 4s   ┌──────────────┐
                                (JWT passed in the socket     │  MongoDB     │
                                 handshake, not the URL)      │  users       │
                                                               │  livelogs    │
                                                               └──────────────┘
                       ┌─────────────────────────┐
                       │  sockets/liveData.js     │
                       │  - simulated/real sensor  │
                       │    feed (swap in real     │
                       │    hardware readings here)│
                       │  - broadcasts to all       │
                       │    connected sockets       │
                       │  - persists every tick     │
                       └─────────────────────────┘
```

**Flow:**
1. User signs up or logs in via REST (`/api/auth/*`). Password is hashed
   with bcrypt; a JWT containing `{ userId, shortName }` is returned.
2. The browser stores the JWT in `localStorage`, then opens a Socket.io
   connection, passing the JWT in the socket handshake (`auth: { token }`)
   — not as a query string, so it doesn't leak into server logs/URLs.
3. The server's `sockets/liveData.js` ticks every 4 seconds, generates the
   next value for each registered metric (a bounded random walk, standing
   in for real sensor input), broadcasts it to every connected client via
   the `liveData` event, and writes it to MongoDB for history.
4. The client updates metric cards, moves/recolors Google Map markers, and
   runs reminder rules (data-driven and time-based) against the new values.
5. REST is still used for anything that isn't a live stream: initial page
   load snapshot (`GET /api/live/current`), historical charts
   (`GET /api/live/history`), and updating a user's reminder thresholds
   (`PATCH /api/live/reminder-preferences`).

**Why WebSockets over polling:** levels can change every few seconds; a
persistent Socket.io connection pushes updates the instant they're
generated, rather than the client repeatedly asking "anything new yet?".

---

## 2. Database schema

### `users` collection (`server/models/User.js`)

| Field | Type | Notes |
|---|---|---|
| `fullName` | String | required |
| `shortName` | String | required — used in the dashboard greeting, e.g. "Vishwa" |
| `email` | String | required, unique, lowercased |
| `passwordHash` | String | required — bcrypt hash, **never** the raw password |
| `location.lat` / `location.lng` / `location.label` | Number/Number/String | optional default map center |
| `reminderPreferences.morningRoutineTime` | String `"HH:MM"` | default `"07:30"` |
| `reminderPreferences.eveningRoutineTime` | String `"HH:MM"` | default `"21:30"` |
| `reminderPreferences.uvAlertThreshold` | Number | default `6` |
| `reminderPreferences.tankLowThreshold` | Number | default `20` |
| `createdAt` / `updatedAt` | Date | via `{ timestamps: true }` |

### `livelogs` collection (`server/models/LiveLog.js`)

| Field | Type | Notes |
|---|---|---|
| `metricType` | String | e.g. `"uv"`, `"airQuality"`, `"tankLevel"` — add new types freely |
| `label` | String | human-readable name shown in the UI |
| `value` | Number | the reading |
| `unit` | String | e.g. `"%"`, `"AQI"` |
| `location.lat` / `location.lng` / `location.label` | Number/Number/String | where this reading came from |
| `timestamp` | Date | indexed; auto-expires after 30 days (TTL index) |

This is intentionally generic — a single collection holds every metric
type, so adding a new sensor/level is a one-line addition to the `METRICS`
array in `sockets/liveData.js`, not a schema migration.

---

## 3. Project structure

```
glow-monitor/
├── server/
│   ├── server.js                # Express + Socket.io entry point
│   ├── package.json
│   ├── .env.example              # copy to .env and fill in
│   ├── models/
│   │   ├── User.js
│   │   └── LiveLog.js
│   ├── routes/
│   │   ├── auth.js               # signup / login
│   │   └── live.js               # current snapshot, history, preferences
│   ├── middleware/
│   │   └── auth.js               # JWT verification (REST + socket)
│   └── sockets/
│       └── liveData.js           # the real-time engine
└── client/
    └── public/
        └── index.html             # single-file dashboard (Tailwind-free vanilla CSS + JS)
```

---

## 4. Running it locally

### Backend
```bash
cd server
cp .env.example .env      # then edit .env: set MONGODB_URI and JWT_SECRET
npm install
npm run dev                # nodemon, or `npm start` for plain node
```
Requires a MongoDB instance — either local (`mongodb://localhost:27017/glow_monitor`)
or a free MongoDB Atlas cluster (paste its connection string into `.env`).

### Frontend
The dashboard is a single static HTML file — no build step required.
```bash
cd client/public
npx serve .                # or just open index.html directly in a browser
```
Before running, open `index.html` and:
1. Confirm `API_BASE` at the top of the `<script>` matches where your
   server is running (defaults to `http://localhost:4000`).
2. Replace `YOUR_GOOGLE_MAPS_API_KEY` in the final `<script>` tag with a
   real Google Maps JavaScript API key (create one in the
   [Google Cloud Console](https://console.cloud.google.com/), restrict it
   to your domain, and enable the "Maps JavaScript API").

### Try it
1. Open the dashboard, sign up with a full name, short name, email and
   password.
2. You'll land on the dashboard immediately, greeted by your short name.
3. Metric cards and map markers start updating every ~4 seconds as soon as
   the server is running and the socket connects.
4. Set your browser/system clock to your `morningRoutineTime` (default
   07:30) to see a time-based reminder fire, or watch the UV card cross
   your threshold to see a data-driven one.

---

## 5. Extending it

- **Real sensors instead of simulated data:** replace the `randomWalk(...)`
  call inside the `setInterval` in `sockets/liveData.js` with a read from
  your actual hardware/API, keeping the same `{ metricType, value, ... }`
  shape — nothing else needs to change.
- **More metrics:** add an entry to the `METRICS` array in
  `sockets/liveData.js` and a matching entry in `METRIC_META` in
  `index.html` for status thresholds/coloring.
- **More reminder rules:** add a condition inside
  `checkDataDrivenReminders` or `checkTimeBasedReminders` in `index.html`,
  or move rule evaluation server-side and emit a dedicated `reminder`
  socket event if you want reminders to survive a page reload / work
  across devices.
- **Push notifications when the tab is closed:** the current reminders are
  in-tab toasts. True push notifications need a service worker plus the
  Web Push API (and, for mobile, a wrapper app or PWA install) — a
  reasonable next step once this is deployed.
- **Production auth hardening:** add refresh tokens, rate-limit
  `/api/auth/login`, and consider email verification before enabling this
  for real users.

## 6. Bug-fix / hardening changelog

A full sweep was done after the initial build. Real issues found and fixed:

**Backend**
- **CORS was silently broken by default.** `cors({ origin: ['*'] })` (an
  array containing the string `'*'`) does *not* mean "allow all" — only
  the bare string `'*'` does. With `CLIENT_ORIGIN` unset (the documented
  default), every real browser request was being blocked before it ever
  reached a route. This was very likely the actual cause of most
  "integration errors." Fixed in `server.js`.
- **A MongoDB hiccup killed the whole process.** `mongoose.connect()`
  failing called `process.exit(1)`, with no retry. Replaced with an
  exponential-backoff reconnect loop, connection-pool settings, and
  `error`/`disconnected`/`reconnected` listeners — the HTTP/Socket.io
  server now stays up through a DB outage.
- **DB-backed routes could hang instead of failing fast.** Mongoose
  buffers queries while disconnected rather than rejecting them
  immediately. Added a readiness check (`middleware/db.js` /  inline
  checks in `routes/auth.js` and `routes/live.js`) so a request gets an
  honest, immediate 503 instead of hanging for the full timeout —
  ordered *after* basic input validation, so a bad request still gets a
  fast 400 instead of a misleading 503.
- **The live-data tick could leak pending DB operations during an
  outage.** `sockets/liveData.js` unconditionally called
  `LiveLog.insertMany()` every 4 seconds even while disconnected; each
  buffered call stacked on the last. It now skips the DB write (but keeps
  emitting live data over the socket) while disconnected.
- **Signup had a race condition and swallowed real errors.** Two
  concurrent signups with the same email could both pass the
  pre-check and then collide on MongoDB's unique index — that
  duplicate-key error, and Mongoose `ValidationError`s, were both falling
  through to a generic, unhelpful 500. Both are now mapped to proper
  409/400 responses.
- **`/api/live/history?hours=...` accepted garbage input.** A
  non-numeric `hours` value produced `NaN` → `Invalid Date`, silently
  passed into the MongoDB query. Now validated and clamped.
- **No process-level safety net.** Added `unhandledRejection` /
  `uncaughtException` handlers (log, don't crash) and graceful shutdown
  on `SIGINT`/`SIGTERM`.
- Unmatched `/api/*` routes previously fell through to Express's default
  HTML 404 page instead of JSON — inconsistent for an API. Fixed.

**Frontend**
- **No fallback if Google Maps fails to load.** An invalid/missing API
  key, network block, or ad blocker left `#map` a permanently blank box
  with zero explanation. Added `window.gm_authFailure`, a try/catch
  around initialization, and a timeout-based fallback message.
- **Reminder timing had a drift bug.** Time-based reminders compared an
  exact `"HH:MM"` string once every 60 seconds — if the interval drifted
  past the target minute, that day's reminder was silently skipped
  entirely. Replaced with a minutes-since-midnight "has it passed the
  target yet" check.
- **Metric cards and the map had no loading state.** Before the first
  live tick, the dashboard was just blank with no indication anything
  was working. Added a skeleton state.
- **Fetch/socket failures were sometimes completely silent.** The initial
  REST snapshot swallowed all errors with no log; socket reconnection
  gave no feedback beyond "Disconnected" even while actively retrying.
  Both now surface clearly (console warnings, toasts, a "Reconnecting…"
  state).
- Added `window.onerror` / `unhandledrejection` listeners as the closest
  equivalent to a React error boundary in a plain HTML/JS app — an
  unexpected exception now surfaces as a toast instead of leaving the
  page silently stuck.

All fixes above were verified by actually running the server (with
`node --check` on every file, plus live `curl` regression tests against
each documented scenario), not just reviewed by eye.
