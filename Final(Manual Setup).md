# 🏋️ Gym Slot Booking System — Complete Project Documentation (Manual Setup, No Docker)

**Stack:** React.js · Node.js/Express · PostgreSQL · MongoDB · Redis
**Purpose of this document:** A single, complete reference covering the project structure, database design, API contract, manual local installation/setup, step-by-step build process, and the full end-to-end request pipeline — all services installed and run directly on your machine, no Docker involved.

---

## Table of Contents

1. [Problem Understanding & Scope](#1-problem-understanding--scope)
2. [Complete Project Structure](#2-complete-project-structure)
3. [Architecture Overview](#3-architecture-overview)
4. [Database Design](#4-database-design)
5. [API Contract](#5-api-contract)
6. [Manual Installation & Setup (No Docker)](#6-manual-installation--setup-no-docker)
7. [Step-by-Step Implementation Guide](#7-step-by-step-implementation-guide)
8. [The Concurrency Problem — Core Logic](#8-the-concurrency-problem--core-logic)
9. [Complete End-to-End Pipeline / Flow](#9-complete-end-to-end-pipeline--flow)
10. [Security](#10-security)
11. [Scalability Notes](#11-scalability-notes)
12. [Known Limitations](#12-known-limitations)

---

## 1. Problem Understanding & Scope

Users book one-hour gym slots that each have a fixed capacity of 10. The system must never allow more bookings than capacity permits, even under heavy concurrent load. Users can cancel to free up their spot.

**In scope:**
1. View slots with remaining capacity
2. Book a slot (rejected once full)
3. Cancel a booking (capacity released)

**Key assumptions:**

| # | Assumption | Reasoning |
|---|---|---|
| A1 | A slot is a fixed, date-bound time window (e.g. 6–7 AM on a specific date), not a recurring template. | Capacity is checked per real occurrence. |
| A2 | A user can hold only one active booking per slot. | Prevents hoarding spots. |
| A3 | A user can book multiple different slots on the same day. | Not restricted by the brief. |
| A4 | Cancellation allowed any time before slot start; no penalty logic. | Keeps scope tight. |
| A5 | Auth required to book/cancel; viewing slots is public. | Standard practice + JWT requirement. |
| A6 | No waitlist, admin panel, or payments. | Explicitly out of scope. |
| A7 | Capacity fixed at 10 per slot, not user-configurable yet. | Stated constraint. |

---

## 2. Complete Project Structure

```
gym-slot-booking/
├── backend/
│   ├── src/
│   │   ├── config/
│   │   │   ├── postgres.js            # pg Pool connection
│   │   │   ├── mongo.js               # Mongoose/MongoClient connection
│   │   │   └── redis.js               # Redis client connection
│   │   ├── middleware/
│   │   │   ├── auth.js                # JWT verification
│   │   │   ├── validate.js            # Joi/Zod request validation
│   │   │   ├── rateLimiter.js         # Redis-backed rate limiting
│   │   │   └── errorHandler.js        # Central error -> consistent JSON shape
│   │   ├── controllers/
│   │   │   ├── authController.js
│   │   │   ├── slotController.js
│   │   │   └── bookingController.js
│   │   ├── routes/
│   │   │   ├── authRoutes.js
│   │   │   ├── slotRoutes.js
│   │   │   └── bookingRoutes.js
│   │   ├── models/
│   │   │   ├── postgres/
│   │   │   │   ├── userModel.js       # SQL query functions for users
│   │   │   │   ├── slotModel.js       # SQL query functions for gym_slots
│   │   │   │   └── bookingModel.js    # SQL query functions for bookings
│   │   │   └── mongo/
│   │   │       ├── activityLog.js     # Schema for activity_logs
│   │   │       └── notification.js    # Schema for notification_history
│   │   ├── db/
│   │   │   └── migrations/
│   │   │       ├── 001_create_users.sql
│   │   │       ├── 002_create_gym_slots.sql
│   │   │       ├── 003_create_bookings.sql
│   │   │       └── 004_create_indexes.sql
│   │   ├── seed/
│   │   │   └── seedSlots.js           # Pre-seeds gym_slots (no admin panel in scope)
│   │   ├── utils/
│   │   │   ├── errors.js              # Named error codes (SLOT_FULL, ALREADY_BOOKED...)
│   │   │   └── logger.js
│   │   ├── app.js                     # Express app, middleware wiring
│   │   └── server.js                  # Entry point
│   ├── tests/
│   │   └── concurrency.test.js        # 3-concurrent-requests-for-1-spot test
│   ├── .env.example
│   └── package.json
├── frontend/
│   ├── src/
│   │   ├── api/
│   │   │   └── client.js              # axios/fetch wrapper calling the API contract
│   │   ├── components/
│   │   │   ├── SlotCard.jsx
│   │   │   ├── BookingList.jsx
│   │   │   └── LoginForm.jsx
│   │   ├── pages/
│   │   │   ├── SlotsPage.jsx
│   │   │   ├── MyBookingsPage.jsx
│   │   │   └── LoginPage.jsx
│   │   ├── context/
│   │   │   └── AuthContext.jsx        # Holds JWT + user state
│   │   ├── App.jsx
│   │   └── main.jsx
│   ├── .env.example
│   ├── package.json
│   └── vite.config.js
├── .gitignore
└── README.md
```

No `Dockerfile`, `.dockerignore`, or `docker-compose.yml` — every service (Postgres, Mongo, Redis, backend, frontend) is installed and run directly on your machine instead.

---

## 3. Architecture Overview

```mermaid
flowchart LR
    subgraph Client
        A[React.js SPA]
    end

    subgraph API["API Layer — Node.js + Express"]
        B[Auth Middleware - JWT]
        C[Validation Middleware]
        D[Rate Limiter]
        E[Slot Controller]
        F[Booking Controller]
    end

    subgraph Cache["Redis"]
        G[(Available-Capacity Cache)]
        H[(Rate-Limit Counters)]
    end

    subgraph Primary["PostgreSQL - Source of Truth"]
        I[(users)]
        J[(gym_slots)]
        K[(bookings)]
    end

    subgraph Secondary["MongoDB - Non-Transactional"]
        L[(activity_logs)]
        M[(notification_history)]
    end

    A -->|HTTPS/JSON| B --> C --> D --> E
    D --> F
    E -->|read-through cache| G
    F -->|atomic conditional UPDATE| J
    F --> K
    F -->|invalidate on write| G
    D -->|check/incr| H
    F -.async fire-and-forget.-> L
    F -.async.-> M
```

**Why each store exists:**

| Data | Store | Why |
|---|---|---|
| Users, gym slots, bookings | **PostgreSQL** | Needs strong relational integrity, ACID transactions, and row-level locking — capacity must never go negative or over-book, and that has to be mathematically guaranteed. |
| Activity/audit logs, notification history | **MongoDB** | High write volume, no relational integrity needed, flexible/append-only schema, never blocks the booking-critical path. |
| Hot-read cache, rate-limit counters | **Redis** | Fast in-memory reads for the most-hit endpoint, and short-lived counters for abuse protection. Never the source of truth. |

---

## 4. Database Design

### 4.1 PostgreSQL Schema

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100)  NOT NULL,
    email           VARCHAR(255)  NOT NULL UNIQUE,
    password_hash   VARCHAR(255)  NOT NULL,
    role            VARCHAR(20)   NOT NULL DEFAULT 'member',  -- member | admin
    created_at      TIMESTAMPTZ   NOT NULL DEFAULT now()
);

CREATE TABLE gym_slots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slot_date       DATE          NOT NULL,
    start_time      TIME          NOT NULL,
    end_time        TIME          NOT NULL,
    capacity        SMALLINT      NOT NULL DEFAULT 10,
    booked_count    SMALLINT      NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),
    CONSTRAINT chk_capacity_bounds CHECK (booked_count >= 0 AND booked_count <= capacity),
    CONSTRAINT uq_slot_window UNIQUE (slot_date, start_time, end_time)
);

CREATE TABLE bookings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    slot_id         UUID NOT NULL REFERENCES gym_slots(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'confirmed',  -- confirmed | cancelled
    booked_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    cancelled_at    TIMESTAMPTZ
);

-- One ACTIVE booking per user per slot (partial unique index)
CREATE UNIQUE INDEX uq_active_booking_per_user_slot
    ON bookings (user_id, slot_id)
    WHERE status = 'confirmed';

-- Read/query performance
CREATE INDEX idx_slots_date            ON gym_slots (slot_date);
CREATE INDEX idx_bookings_user_status  ON bookings (user_id, status);
CREATE INDEX idx_bookings_slot_status  ON bookings (slot_id, status);
```

**Design notes:**
- `booked_count` is a denormalized counter on `gym_slots`, kept correct by a single atomic `UPDATE` instead of running `COUNT(*)` on every request.
- The `CHECK` constraint is a database-enforced safety net — even a buggy application can never push `booked_count` past `capacity` or below `0`.
- The partial unique index prevents double-booking without a separate check-then-insert step (which would itself be racy).

### 4.2 MongoDB Collections

```jsonc
// activity_logs
{
  "_id": ObjectId,
  "userId": "uuid-from-postgres",
  "action": "booking_created" | "booking_cancelled",
  "slotId": "uuid-from-postgres",
  "timestamp": ISODate,
  "metadata": { "ip": "...", "userAgent": "..." }
}

// notification_history
{
  "_id": ObjectId,
  "userId": "uuid-from-postgres",
  "channel": "email" | "push",
  "message": "Your 6 AM slot is confirmed",
  "sentAt": ISODate,
  "status": "sent" | "failed"
}
```

These writes are fire-and-forget/async and never sit in the critical booking path — Mongo being briefly unavailable must never block or fail a booking.

### 4.3 Redis Usage

| Use | Key pattern | Strategy |
|---|---|---|
| Cache hot read (available capacity) | `slot:{slotId}:available` | Cache-aside: read Redis first; on miss, read Postgres and populate with a short TTL (~10s). Invalidated on every successful booking/cancellation. |
| Rate limiting `POST /api/bookings` | `ratelimit:booking:{userId}` | Sliding/fixed-window counter (`INCR` + `EXPIRE`), capping attempts per minute; returns `429` past the limit. |

---

## 5. API Contract

| Method | Endpoint | Auth | Request Body | Success | Error Cases |
|---|---|---|---|---|---|
| POST | `/api/auth/register` | ❌ | `{name, email, password}` | `201 {id, name, email}` | `400` validation, `409` email exists |
| POST | `/api/auth/login` | ❌ | `{email, password}` | `200 {token, expiresIn}` | `401` invalid credentials |
| GET | `/api/slots?date=YYYY-MM-DD` | ❌ | — | `200 [{id, startTime, endTime, capacity, available}]` | `400` bad date |
| GET | `/api/slots/:id` | ❌ | — | `200 {id, startTime, endTime, capacity, available}` | `404` not found |
| POST | `/api/bookings` | ✅ | `{slotId}` | `201 {bookingId, slotId, status}` | `400`, `401`, `404`, `409 SLOT_FULL`, `409 ALREADY_BOOKED` |
| DELETE | `/api/bookings/:id` | ✅ | — | `200 {bookingId, status:"cancelled"}` | `401`, `403`, `404`, `409` |
| GET | `/api/bookings/me` | ✅ | — | `200 [{bookingId, slot, status}]` (paginated) | `401` |

**Standard error shape:**
```json
{
  "error": {
    "code": "SLOT_FULL",
    "message": "This slot has no remaining capacity."
  }
}
```

All list endpoints support `?page=&limit=`.

---

## 6. Manual Installation & Setup (No Docker)

Every service — PostgreSQL, MongoDB, Redis, the backend, and the frontend — is installed directly on your machine and connected via `localhost`. This section replaces the Docker approach entirely.

### 6.1 Prerequisites checklist

- Node.js LTS (v18 or v20) + npm
- PostgreSQL (v14+)
- MongoDB (v6+)
- Redis (v7+)
- `git`

### 6.2 Installing PostgreSQL manually

**Windows:**
1. Download the installer from postgresql.org and run it.
2. During setup, set a password for the `postgres` superuser and note the port (default `5432`).
3. The installer also installs **pgAdmin** (GUI) and adds `psql` to your PATH.

**macOS:**
```bash
brew install postgresql@16
brew services start postgresql@16
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

**Create the database and user (all platforms, via `psql`):**
```bash
sudo -u postgres psql
```
```sql
CREATE USER gymuser WITH PASSWORD 'gympassword';
CREATE DATABASE gymbooking OWNER gymuser;
GRANT ALL PRIVILEGES ON DATABASE gymbooking TO gymuser;
\q
```

**Verify it's running:**
```bash
psql -h localhost -U gymuser -d gymbooking -c "SELECT 1;"
```

### 6.3 Installing MongoDB manually

**Windows:** Download "MongoDB Community Server" from mongodb.com, run the installer, and choose to install it as a Windows Service (it will then start automatically).

**macOS:**
```bash
brew tap mongodb/brew
brew install mongodb-community@7.0
brew services start mongodb-community@7.0
```

**Linux (Ubuntu/Debian):**
```bash
# Follow MongoDB's official apt-repository instructions for your Ubuntu version, then:
sudo apt update
sudo apt install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod
```

**Verify it's running:**
```bash
mongosh --eval "db.runCommand({ ping: 1 })"
```
No manual database/collection creation is needed upfront — Mongoose/MongoClient will create `gymbooking` and its collections automatically on first write.

### 6.4 Installing Redis manually

**Windows:** Native Redis isn't officially supported on Windows. Easiest options: install **Memurai** (a Redis-compatible Windows service) or use **WSL2** and follow the Linux steps below inside it.

**macOS:**
```bash
brew install redis
brew services start redis
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt update
sudo apt install redis-server
sudo systemctl start redis-server
sudo systemctl enable redis-server
```

**Verify it's running:**
```bash
redis-cli ping
# should return: PONG
```

### 6.5 Backend setup

```bash
cd gym-slot-booking/backend
npm install
```

Create `backend/.env` (copy from `.env.example` and fill in real local values):
```
DATABASE_URL=postgres://gymuser:gympassword@localhost:5432/gymbooking
MONGO_URL=mongodb://localhost:27017/gymbooking
REDIS_URL=redis://localhost:6379
JWT_SECRET=replace-with-a-long-random-string
PORT=5000
```

Run the Postgres migrations (via your migration tool, or by piping the SQL files directly):
```bash
psql -h localhost -U gymuser -d gymbooking -f src/db/migrations/001_create_users.sql
psql -h localhost -U gymuser -d gymbooking -f src/db/migrations/002_create_gym_slots.sql
psql -h localhost -U gymuser -d gymbooking -f src/db/migrations/003_create_bookings.sql
psql -h localhost -U gymuser -d gymbooking -f src/db/migrations/004_create_indexes.sql
```

Seed a few slots:
```bash
node src/seed/seedSlots.js
```

Start the backend in dev mode:
```bash
npm run dev
# (nodemon watches src/ and restarts on save — runs on http://localhost:5000)
```

### 6.6 Frontend setup

```bash
cd gym-slot-booking/frontend
npm install
```

Create `frontend/.env`:
```
VITE_API_BASE_URL=http://localhost:5000/api
```

Start the frontend dev server:
```bash
npm run dev
# runs on http://localhost:5173 (Vite default) or similar
```

### 6.7 Verifying all connections end-to-end

1. `redis-cli ping` → `PONG`
2. `psql -h localhost -U gymuser -d gymbooking -c "\dt"` → shows `users`, `gym_slots`, `bookings`
3. `mongosh --eval "db.getMongo().getDBNames()"` → shows `gymbooking` once the backend has written to it
4. Hit `http://localhost:5000/api/slots` in a browser or `curl` → should return your seeded slots as JSON
5. Open `http://localhost:5173` → the React app should load and display those slots

### 6.8 Common issues when running manually

| Symptom | Likely cause | Fix |
|---|---|---|
| Backend crashes on start with `ECONNREFUSED 5432` | Postgres service not running | Start it (`brew services start postgresql@16` / `sudo systemctl start postgresql`) |
| Backend crashes with Mongo connection error | Mongo service not running, or wrong `MONGO_URL` | Confirm `mongosh` connects; check `.env` |
| `redis-cli ping` fails | Redis not started, or Windows without WSL/Memurai | Start the service, or install Memurai/WSL2 on Windows |
| Frontend calls fail with CORS errors | Backend `cors()` not allowing the frontend's port | Confirm `cors` middleware allows `http://localhost:5173` |
| `permission denied for database gymbooking` | User wasn't granted privileges correctly | Re-run the `GRANT ALL PRIVILEGES` step in §6.2 |

### 6.9 Running everything together (day-to-day)

Since there's no single "up" command without Docker, you'll typically run four things in separate terminal tabs (or use a process manager like `concurrently`/`pm2` to combine them):

```bash
# Terminal 1 — Postgres, Mongo, Redis are usually background services
# once installed (see 6.2–6.4), so nothing to run here unless they've stopped.

# Terminal 2
cd backend && npm run dev

# Terminal 3
cd frontend && npm run dev
```

Optional: add a root-level `package.json` with a `concurrently` script so `npm run dev` at the repo root starts both backend and frontend together:
```json
{
  "scripts": {
    "dev": "concurrently \"npm run dev --prefix backend\" \"npm run dev --prefix frontend\""
  },
  "devDependencies": {
    "concurrently": "^8.2.2"
  }
}
```

---

## 7. Step-by-Step Implementation Guide

### Step 1 — Repo & environment scaffolding
1. `mkdir gym-slot-booking && cd gym-slot-booking && git init`
2. Create `backend/` and `frontend/` folders.
3. In `backend/`: `npm init -y`, then install:
   - `express`, `pg`, `mongoose`, `ioredis`, `jsonwebtoken`, `bcrypt`, `joi`, `dotenv`, `cors`
   - Dev: `nodemon`
4. Create `backend/.env.example` with variable names only: `DATABASE_URL`, `MONGO_URL`, `REDIS_URL`, `JWT_SECRET`, `PORT`.
5. Add `.env` to `.gitignore` (root level, covers both backend and frontend).
6. Install Postgres, MongoDB, and Redis locally per §6.2–§6.4 before writing any app code, so you can develop against real running services from day one.

### Step 2 — PostgreSQL schema (source of truth)
1. Confirm Postgres is running locally (§6.2).
2. Add migration files under `db/migrations/` containing the schema from §4.1: `users`, `gym_slots`, `bookings`, all constraints and indexes.
3. Run each migration file with `psql -f` (§6.5) and verify tables exist with `\dt` in `psql`.
4. Run the seed script to pre-populate a few `gym_slots` rows (no admin panel in scope, per A6).

### Step 3 — MongoDB collections
1. Confirm Mongo is running locally (§6.3).
2. Connect from `config/mongo.js` using the local `MONGO_URL`.
3. Define lightweight schemas for `activity_logs` and `notification_history` from §4.2 — for documentation/structure, not strict enforcement.

### Step 4 — Redis connection
1. Confirm Redis is running locally (§6.4) — `redis-cli ping` should return `PONG`.
2. Connect from `config/redis.js`.
3. No schema needed — just the two key patterns from §4.3.

### Step 5 — Auth (JWT)
1. `POST /api/auth/register` — hash password with bcrypt (cost ≥ 10), insert into `users`.
2. `POST /api/auth/login` — verify password, issue JWT with short expiry.
3. `auth.js` middleware — verifies `Authorization: Bearer <token>` on protected routes.

### Step 6 — Slot endpoints (public reads)
1. `GET /api/slots?date=YYYY-MM-DD` — cache-aside against Redis, falling back to Postgres on a cache miss or Redis outage.
2. `GET /api/slots/:id` — same pattern for a single slot.
3. Add pagination (`?page=&limit=`).

### Step 7 — Booking endpoint (the core concurrency requirement)
Implement exactly the atomic pattern from §8 — no separate `SELECT` then `INSERT`.
1. Wrap in a Postgres transaction using a single client from the pool.
2. On the unique-violation from `uq_active_booking_per_user_slot`, translate to `409 ALREADY_BOOKED`.
3. On success: invalidate the Redis cache key for that slot, then fire an async write to `activity_logs` in Mongo.
4. Apply the rate-limiter middleware to this route.

### Step 8 — Cancellation endpoint
Implement the symmetric atomic pair from §8.
- `403` if requester isn't the booking owner.
- `409` if already cancelled.
- Invalidate the Redis cache key on success.

### Step 9 — `GET /api/bookings/me`
Paginated list of the authenticated user's bookings, joined with slot info.

### Step 10 — Error handling middleware
Central Express error handler mapping every failure to the standard shape (§5). Cover: Postgres down → `503`, Redis down → fall back to Postgres / fail-open, Mongo down → swallow + log warning, invalid input → `400` at validation layer.

### Step 11 — Input validation
Joi/Zod schemas for every request body and query param, applied before controllers run.

### Step 12 — Frontend (React)
1. Login/Register page → stores JWT.
2. Slots page → lists slots with remaining capacity, "Book" button per slot.
3. My Bookings page → lists the user's bookings with a "Cancel" button.
4. `api/client.js` wrapping fetch/axios calls, attaching the JWT header for protected calls, pointing at `http://localhost:5000/api` via `VITE_API_BASE_URL`.

### Step 13 — Run the full stack locally
1. Confirm Postgres, Mongo, Redis are all running as background services (§6.2–6.4).
2. Start the backend: `cd backend && npm run dev`.
3. Start the frontend: `cd frontend && npm run dev`.
4. Confirm the frontend can reach the backend, and the backend can reach all three data stores (§6.7).

### Step 14 — Test the concurrency scenario
This is the most important validation step.
1. Seed a slot with `booked_count = 9`, `capacity = 10` (1 spot left).
2. Fire 3 concurrent `POST /api/bookings` requests for that slot (`Promise.all` in a script, or `autocannon`/`k6`).
3. Confirm exactly 1 succeeds (`201`), the other 2 get `409 SLOT_FULL`, and `booked_count` never exceeds `10`.

### Step 15 — Documentation & submission
Document in the repo's own `README.md`:
- Env vars needed
- How to install and start Postgres/Mongo/Redis manually (§6.2–6.4)
- How to run migrations
- How to start backend (`npm run dev`) and frontend (`npm run dev`)

---

## 8. The Concurrency Problem — Core Logic

**Scenario:** 1 spot left, 3 requests arrive within the same second → exactly 1 must succeed, capacity must never go negative or exceed 10.

### Chosen approach: single atomic conditional `UPDATE`

```sql
BEGIN;

UPDATE gym_slots
SET booked_count = booked_count + 1
WHERE id = $1
  AND booked_count < capacity
RETURNING booked_count;

-- 0 rows returned -> slot was already full -> rollback, return 409 SLOT_FULL
-- 1 row affected  -> this request "won" the last spot -> proceed to insert

INSERT INTO bookings (user_id, slot_id, status)
VALUES ($2, $1, 'confirmed');

COMMIT;
```

**Why this beats "read count, then insert if < capacity":** reading and writing as two separate steps is a classic TOCTOU (time-of-check-to-time-of-use) race — three concurrent requests could all read `9/10`, all decide "there's room," and all insert, blowing past capacity. The `UPDATE ... WHERE booked_count < capacity` is a single atomic statement: PostgreSQL serializes concurrent writers to the same row at the row level, so only as many `UPDATE`s as there are remaining spots can ever succeed — the rest affect 0 rows and are rejected deterministically. No explicit `SELECT ... FOR UPDATE` lock is needed because the conditional `UPDATE` *is* the atomic check-and-act.

**Why not a Redis distributed lock?** Redis isn't the source of truth — Postgres is. A Redis lock (e.g. Redlock) adds a second system that can itself fail, drift, or expire mid-request. Using the database's own atomicity guarantees is simpler and has one fewer moving part to reason about under failure. Redis is reserved for what it's actually good at: caching hot reads and rate-limiting — not enforcing a hard business invariant.

**Double-booking protection:** `uq_active_booking_per_user_slot` means even a double-click on "Book" fails on the second `INSERT` with a unique-violation, translated to `409 ALREADY_BOOKED`.

**Cancellation (symmetric, also atomic):**
```sql
BEGIN;
UPDATE bookings SET status = 'cancelled', cancelled_at = now()
WHERE id = $1 AND user_id = $2 AND status = 'confirmed';

UPDATE gym_slots SET booked_count = booked_count - 1
WHERE id = $3 AND booked_count > 0;
COMMIT;
```

---

## 9. Complete End-to-End Pipeline / Flow

### 9.1 High-level request flow (booking a slot)

```
1. React SPA
      └─ User clicks "Book" on a slot card
            ↓
2. Frontend api/client.js
      └─ POST /api/bookings  { slotId }
      └─ Attaches "Authorization: Bearer <JWT>" header
            ↓
3. Express app.js — middleware chain
      └─ CORS check
      └─ auth.js — verifies JWT, attaches req.user
      └─ validate.js — checks slotId is a valid UUID (Joi/Zod)
      └─ rateLimiter.js — INCR ratelimit:booking:{userId} in Redis,
         reject with 429 if over the per-minute cap
            ↓
4. bookingController.js
      └─ Opens a Postgres client from the pool, BEGIN transaction
      └─ Atomic conditional UPDATE gym_slots
         SET booked_count = booked_count + 1
         WHERE id = slotId AND booked_count < capacity
      └─ 0 rows affected?  → ROLLBACK → throw SLOT_FULL (409)
      └─ 1 row affected?   → INSERT INTO bookings (...)
         (unique-violation on repeat booking → ALREADY_BOOKED, 409)
      └─ COMMIT
            ↓
5. Post-commit side effects (do not block the response)
      └─ DEL slot:{slotId}:available  in Redis  (cache invalidation)
      └─ Async fire-and-forget write to Mongo activity_logs
         { userId, action: "booking_created", slotId, timestamp }
      └─ Async fire-and-forget write to Mongo notification_history
         (e.g. queue a confirmation email/push)
            ↓
6. errorHandler.js (only if something threw)
      └─ Maps any thrown error to the standard JSON error shape
         { error: { code, message } } with the correct HTTP status
            ↓
7. Response returned to client
      └─ 201 { bookingId, slotId, status: "confirmed" }
         or 409 { error: { code: "SLOT_FULL", ... } }
            ↓
8. React SPA
      └─ Updates local state / re-fetches slot list
      └─ Shows success toast, or the specific error message
```

### 9.2 High-level request flow (viewing slots — the hot read path)

```
1. React SPA requests GET /api/slots?date=YYYY-MM-DD
            ↓
2. Express: CORS → validate.js (checks date format) → slotController.js
            ↓
3. slotController.js — cache-aside pattern
      └─ GET slot:{slotId}:available for each candidate slot (or a
         batched key covering the date) from Redis
      └─ Cache HIT  → build response directly from cached values
      └─ Cache MISS → query Postgres (gym_slots WHERE slot_date = $1),
         compute "available = capacity - booked_count",
         SET the Redis key with a short TTL (~10s), then respond
      └─ If Redis is down entirely → skip cache, read Postgres directly
         (fail-open on cache, never fail-closed on this read path)
            ↓
4. Response: 200 [{ id, startTime, endTime, capacity, available }, ...]
```

### 9.3 Full system lifecycle (local machine, no Docker)

```
1. Postgres, MongoDB, Redis installed as native services
   (Homebrew / apt / Windows installer — see §6.2–6.4)
      └─ Each starts as a background OS service, independent of
         your project — they keep running across reboots (if you
         enabled autostart) or need a manual "start service" command
        ↓
2. Postgres migrations run once via psql (§6.5)
      └─ users, gym_slots, bookings tables + constraints + indexes
        ↓
3. Backend started manually: `cd backend && npm run dev`
      └─ Reads DATABASE_URL / MONGO_URL / REDIS_URL from .env,
         all pointing at localhost with each service's default port
      └─ Connects to Postgres, Mongo, Redis
      └─ Express server listens on :5000
        ↓
4. Frontend started manually: `cd frontend && npm run dev`
      └─ Vite dev server serves the React app on :5173
      └─ Calls http://localhost:5000/api/... for data
        ↓
5. End user hits http://localhost:5173
      └─ React SPA loads, full request pipeline (§9.1 / §9.2)
         executes per API call
```

### 9.4 Failure-path pipeline (what happens when a dependency is down)

```
PostgreSQL unreachable (service stopped)
      → booking/cancel/register/login endpoints fail fast with 503
      → API never silently "pretends" success
      → Fix: start the Postgres service again
        (brew services start postgresql@16 / sudo systemctl start postgresql)

Redis unreachable (service stopped)
      → cache reads fall back to Postgres directly (slower, still correct)
      → rate limiter fails open (or fails closed on the booking route
        specifically, as a deliberate documented trade-off)
      → Fix: start Redis again (brew services start redis /
        sudo systemctl start redis-server)

MongoDB unreachable (service stopped)
      → activity_logs / notification_history writes are best-effort
      → failure is caught, logged as a warning, and swallowed
      → booking transaction has already committed in Postgres by this
        point, so the booking itself is never rolled back or blocked
      → Fix: start Mongo again (brew services start mongodb-community@7.0
        / sudo systemctl start mongod)
```

---

## 10. Security

- **AuthN:** JWT issued on login, short expiry + refresh strategy; `Authorization: Bearer <token>` required on booking/cancel routes.
- **AuthZ:** a user can only cancel their own booking (`403` otherwise); `admin` role reserved for future slot-management endpoints (out of scope now).
- **Passwords:** hashed with bcrypt (cost factor ≥ 10), never stored or logged in plain text.
- **Input validation/sanitization:** schema validation (Joi/Zod) on every request before it reaches business logic — blocks SQL injection (also mitigated structurally by parameterized queries) and NoSQL injection into Mongo writes.
- **Secrets:** DB URLs, JWT secret, Redis URL all in environment variables, never hard-coded or committed (`.env` gitignored, `.env.example` committed).
- **Rate-limiting:** applied to `POST /api/bookings` and `POST /api/auth/login` (to blunt brute-force attempts).
- **Transport:** HTTPS assumed in front of the API (terminated at load balancer/reverse proxy in production).

---

## 11. Scalability Notes

- **Indexing:** every filtered/sorted column (`slot_date`, `user_id`, `slot_id`, `status`) is indexed; no endpoint does a full table scan.
- **Pagination:** enforced on all list endpoints; no unbounded queries.
- **Caching:** the hottest reads (slot list, single-slot availability) are cached in Redis with short TTL + active invalidation.
- **Scaling to 100x traffic:**
  - Node/Express API servers are stateless (JWT, no server-side session) → horizontally scale behind a load balancer.
  - PostgreSQL: connection pooling (PgBouncer) for many app instances; read replicas for `GET /api/slots` reads, while writes stay on the primary for the atomicity guarantee.
  - Redis: can move to a small cluster/replica setup without affecting correctness, since it's not the system of record.
  - MongoDB: scales horizontally by sharding on `userId`/date if audit-log volume grows, with zero impact on the booking-critical path (fully decoupled, async writes).

---

## 12. Known Limitations

- No waitlist / notify-when-available feature — intentionally excluded.
- No admin UI for creating/editing slots — slots are pre-seeded via migration/seed script.
- Fixed capacity (10), not per-slot-configurable in this iteration — schema already supports arbitrary capacities if that changes.
- Rate-limiter fail-open vs fail-closed during a Redis outage is a real, documented trade-off.
- No idempotency key on `POST /api/bookings` yet for client retry-on-timeout safety — the unique-active-booking constraint prevents a *duplicate* booking, but a true idempotency-key pattern would be a cleaner long-term fix.
- No containerization in this version — every developer/environment must install Postgres, MongoDB, and Redis natively, which is more setup friction than a single `docker compose up` and makes "works on my machine" environment drift more likely.
