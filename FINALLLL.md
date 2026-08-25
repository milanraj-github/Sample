# 🏋️ Gym Slot Booking System — Complete Project Documentation

**Stack:** React.js · Node.js/Express · PostgreSQL · MongoDB · Redis · Docker
**Purpose of this document:** A single, complete reference covering the project structure, database design, API contract, Docker setup, step-by-step build process, and the full end-to-end request pipeline — consolidated from the design document and the development guide into one detailed source of truth.

---

## Table of Contents

1. [Problem Understanding & Scope](#1-problem-understanding--scope)
2. [Complete Project Structure](#2-complete-project-structure)
3. [Architecture Overview](#3-architecture-overview)
4. [Database Design](#4-database-design)
5. [API Contract](#5-api-contract)
6. [Docker Setup (Full Detail)](#6-docker-setup-full-detail)
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
│   ├── .dockerignore
│   ├── package.json
│   └── Dockerfile
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
│   ├── .dockerignore
│   ├── package.json
│   ├── vite.config.js
│   └── Dockerfile
├── docker-compose.yml                 # Postgres + Mongo + Redis + backend + frontend
├── .gitignore
└── README.md
```

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

## 6. Docker Setup (Full Detail)

Since you're using Docker, here's the complete containerization for local dev and a one-command spin-up of the whole stack.

### 6.1 `backend/Dockerfile`

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --production

COPY . .

EXPOSE 5000

CMD ["node", "src/server.js"]
```

For local development with hot reload, use a second stage or a dev-specific compose override running `nodemon` instead of `node`.

### 6.2 `frontend/Dockerfile`

```dockerfile
# Build stage
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Serve stage
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 6.3 `.dockerignore` (both backend and frontend)

```
node_modules
npm-debug.log
.env
.git
dist
```

### 6.4 `docker-compose.yml` (root of the repo)

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:16-alpine
    container_name: gym-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: gymuser
      POSTGRES_PASSWORD: gympassword
      POSTGRES_DB: gymbooking
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./backend/src/db/migrations:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U gymuser -d gymbooking"]
      interval: 5s
      timeout: 5s
      retries: 5

  mongo:
    image: mongo:7
    container_name: gym-mongo
    restart: unless-stopped
    ports:
      - "27017:27017"
    volumes:
      - mongodata:/data/db

  redis:
    image: redis:7-alpine
    container_name: gym-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redisdata:/data

  backend:
    build: ./backend
    container_name: gym-backend
    restart: unless-stopped
    ports:
      - "5000:5000"
    environment:
      DATABASE_URL: postgres://gymuser:gympassword@postgres:5432/gymbooking
      MONGO_URL: mongodb://mongo:27017/gymbooking
      REDIS_URL: redis://redis:6379
      JWT_SECRET: your-jwt-secret-here
      PORT: 5000
    depends_on:
      postgres:
        condition: service_healthy
      mongo:
        condition: service_started
      redis:
        condition: service_started

  frontend:
    build: ./frontend
    container_name: gym-frontend
    restart: unless-stopped
    ports:
      - "3000:80"
    depends_on:
      - backend

volumes:
  pgdata:
  mongodata:
  redisdata:
```

**Notes on this setup:**
- `postgres` mounts the migration folder into `docker-entrypoint-initdb.d`, so the schema is created automatically the first time the container starts with a fresh volume.
- `backend` uses `depends_on` with a health check on Postgres so it doesn't start accepting traffic before the database is actually ready — not just "container started."
- Internal service-to-service networking uses the Compose service names (`postgres`, `mongo`, `redis`) as hostnames — Docker Compose sets up a shared network automatically.
- Never bake real secrets (`JWT_SECRET`, DB passwords) into the compose file for anything beyond local dev — use a `.env` file at the repo root (Compose reads it automatically) and keep it gitignored.

### 6.5 Root-level `.env` for Docker Compose (gitignored)

```
POSTGRES_USER=gymuser
POSTGRES_PASSWORD=gympassword
POSTGRES_DB=gymbooking
JWT_SECRET=replace-with-a-long-random-string
```

Reference these in `docker-compose.yml` with `${POSTGRES_USER}` syntax instead of hardcoding, once you move past local experimentation.

### 6.6 Common Docker commands for this project

```bash
# Build and start everything
docker compose up --build

# Start in detached mode
docker compose up -d

# View logs for one service
docker compose logs -f backend

# Run a one-off command inside the backend container (e.g. a migration script)
docker compose exec backend npm run migrate

# Stop everything but keep volumes (data persists)
docker compose down

# Stop and wipe all data (fresh Postgres/Mongo/Redis)
docker compose down -v

# Rebuild a single service after a Dockerfile change
docker compose build backend && docker compose up -d backend
```

### 6.7 Local dev vs Docker dev

You don't have to run everything in Docker while actively coding the backend/frontend. A common pattern:
- Run **only** `postgres`, `mongo`, `redis` via Docker (`docker compose up postgres mongo redis`).
- Run `backend` and `frontend` natively on your machine with `npm run dev` for fast hot-reload, pointing their `.env` files at `localhost:5432`, `localhost:27017`, `localhost:6379`.
- Once things are stable, bring up the full `docker compose up --build` stack to prove it works end-to-end in containers — this is what you'd actually deploy.

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
6. Write the `Dockerfile`s and root `docker-compose.yml` from §6 now, even before writing app code — this lets you spin up Postgres/Mongo/Redis immediately with `docker compose up postgres mongo redis` and develop against real services from day one.

### Step 2 — PostgreSQL schema (source of truth)
1. Start Postgres via Docker: `docker compose up -d postgres`.
2. Add migration files under `db/migrations/` containing the schema from §4.1: `users`, `gym_slots`, `bookings`, all constraints and indexes.
3. Run migrations against the containerized Postgres instance (via a migration tool, or the auto-run `docker-entrypoint-initdb.d` mechanism on first container start) and verify tables exist with `docker compose exec postgres psql -U gymuser -d gymbooking -c '\dt'`.
4. Run the seed script to pre-populate a few `gym_slots` rows (no admin panel in scope, per A6).

### Step 3 — MongoDB collections
1. Start Mongo via Docker: `docker compose up -d mongo`.
2. Connect from `config/mongo.js`.
3. Define lightweight schemas for `activity_logs` and `notification_history` from §4.2 — for documentation/structure, not strict enforcement.

### Step 4 — Redis connection
1. Start Redis via Docker: `docker compose up -d redis`.
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
4. `api/client.js` wrapping fetch/axios calls, attaching the JWT header for protected calls, pointing at the backend's URL (via an env var so it works both in local dev and inside Docker).

### Step 13 — Containerize and run the full stack
1. Write/confirm `backend/Dockerfile` and `frontend/Dockerfile` (§6.1–6.2).
2. From the repo root: `docker compose up --build`.
3. Confirm the frontend (port `3000`) can reach the backend (port `5000`), and the backend can reach Postgres/Mongo/Redis using the Compose service hostnames.

### Step 14 — Test the concurrency scenario
This is the most important validation step.
1. Seed a slot with `booked_count = 9`, `capacity = 10` (1 spot left).
2. Fire 3 concurrent `POST /api/bookings` requests for that slot (`Promise.all` in a script, or `autocannon`/`k6`).
3. Confirm exactly 1 succeeds (`201`), the other 2 get `409 SLOT_FULL`, and `booked_count` never exceeds `10`.

### Step 15 — Documentation & submission
Document in the repo's own `README.md`:
- Env vars needed
- How to run migrations
- How to start everything with `docker compose up --build`
- How to start backend/frontend natively for dev (`npm run dev`)

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

### 9.3 Full system lifecycle (deployment view, with Docker)

```
docker compose up --build
        ↓
1. postgres container starts
      └─ Runs migration SQL from db/migrations/ on first boot
         (via docker-entrypoint-initdb.d mount)
      └─ Healthcheck passes once pg_isready succeeds
        ↓
2. mongo + redis containers start (no schema step needed)
        ↓
3. backend container starts (waits on postgres healthcheck via depends_on)
      └─ Connects to Postgres, Mongo, Redis using Compose service
         hostnames (postgres, mongo, redis) from env vars
      └─ Express server listens on :5000
        ↓
4. frontend container starts (waits on backend)
      └─ Nginx serves the built React static files on :80,
         mapped to host port :3000
        ↓
5. End user hits http://localhost:3000
      └─ React SPA loads, calls http://localhost:5000/api/... for data
      └─ Full request pipeline (§9.1 / §9.2) executes per API call
```

### 9.4 Failure-path pipeline (what happens when a dependency is down)

```
PostgreSQL unreachable
      → booking/cancel/register/login endpoints fail fast with 503
      → API never silently "pretends" success

Redis unreachable
      → cache reads fall back to Postgres directly (slower, still correct)
      → rate limiter fails open (or fails closed on the booking route
        specifically, as a deliberate documented trade-off)

MongoDB unreachable
      → activity_logs / notification_history writes are best-effort
      → failure is caught, logged as a warning, and swallowed
      → booking transaction has already committed in Postgres by this
        point, so the booking itself is never rolled back or blocked
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
