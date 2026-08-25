# 🏋️ Gym Slot Booking — Development & Build Guide

**Phase:** Day 2 — Implementation (matches the Day 1 Design Document exactly)
**Stack:** React.js · Node.js/Express · PostgreSQL · MongoDB · Redis (MERN + SQL)

> This guide only covers what's in the approved design (`README.md`). No feature, endpoint, or table beyond what's listed there is included — per the assignment rule of "no additional features."

---

## 1. Project Structure

A single repo, split into `backend/` and `frontend/`, is enough for this scope — no need for a monorepo tool.

```
gym-slot-booking/
├── backend/
│   ├── src/
│   │   ├── config/
│   │   │   ├── postgres.js        # pg Pool connection
│   │   │   ├── mongo.js           # Mongoose/MongoClient connection
│   │   │   └── redis.js           # Redis client connection
│   │   ├── middleware/
│   │   │   ├── auth.js            # JWT verification
│   │   │   ├── validate.js        # Joi/Zod request validation
│   │   │   ├── rateLimiter.js     # Redis-backed rate limiting
│   │   │   └── errorHandler.js    # Central error -> consistent JSON shape
│   │   ├── controllers/
│   │   │   ├── authController.js
│   │   │   ├── slotController.js
│   │   │   └── bookingController.js
│   │   ├── routes/
│   │   │   ├── authRoutes.js
│   │   │   ├── slotRoutes.js
│   │   │   └── bookingRoutes.js
│   │   ├── models/
│   │   │   ├── postgres/          # SQL query functions for users, gym_slots, bookings
│   │   │   └── mongo/             # Schemas for activity_logs, notification_history
│   │   ├── db/
│   │   │   └── migrations/        # SQL migration files (schema from §3.2)
│   │   ├── utils/
│   │   │   └── errors.js          # Named error codes (SLOT_FULL, ALREADY_BOOKED, etc.)
│   │   ├── app.js                 # Express app, middleware wiring
│   │   └── server.js              # Entry point
│   ├── .env.example
│   ├── package.json
│   └── Dockerfile (optional)
├── frontend/
│   ├── src/
│   │   ├── api/                   # axios/fetch wrapper calling the API contract
│   │   ├── components/            # SlotCard, BookingList, LoginForm
│   │   ├── pages/                 # SlotsPage, MyBookingsPage, LoginPage
│   │   └── App.jsx
│   ├── package.json
│   └── Dockerfile (optional)
├── docker-compose.yml (optional — Postgres + Redis + Mongo)
└── README.md
```

---

## 2. Prerequisites

- Node.js LTS + npm
- PostgreSQL running locally (or via Docker)
- MongoDB running locally (or via Docker)
- Redis running locally (or via Docker)
- `git`

---

## 3. Step-by-Step Build Procedure

### Step 1 — Repo & environment scaffolding
1. `mkdir gym-slot-booking && cd gym-slot-booking && git init`
2. Create `backend/` and `frontend/` folders.
3. In `backend/`: `npm init -y`, then install:
   - `express`, `pg`, `mongoose` (or `mongodb`), `ioredis` (or `redis`), `jsonwebtoken`, `bcrypt`, `joi` (or `zod`), `dotenv`, `cors`
   - Dev: `nodemon`
4. Create `.env.example` with the variable names only (no real secrets), matching §7 of the design: `DATABASE_URL`, `MONGO_URL`, `REDIS_URL`, `JWT_SECRET`, `PORT`.
5. Add `.env` to `.gitignore`.

### Step 2 — PostgreSQL schema (source of truth)
1. Create the database.
2. Add a migration file under `db/migrations/` containing exactly the schema from the design doc §3.2:
   - `users`, `gym_slots`, `bookings` tables
   - `chk_capacity_bounds` CHECK constraint
   - `uq_slot_window` unique constraint
   - `uq_active_booking_per_user_slot` partial unique index
   - Indexes: `idx_slots_date`, `idx_bookings_user_status`, `idx_bookings_slot_status`
3. Run the migration against your local Postgres instance and verify the tables exist.
4. Seed a few `gym_slots` rows manually (since there's no admin panel in scope — slots are pre-seeded per assumption A6/design §10).

### Step 3 — MongoDB collections
1. Connect to Mongo from `config/mongo.js`.
2. Define the two collections from §3.3: `activity_logs` and `notification_history`. No strict schema enforcement needed (Mongo is intentionally flexible here) — a lightweight Mongoose schema is fine for structure/documentation purposes.

### Step 4 — Redis connection
1. Connect to Redis from `config/redis.js`.
2. No schema needed — Redis is used only for:
   - Cache-aside key `slot:{slotId}:available` (§6)
   - Rate-limit counter key `ratelimit:booking:{userId}` (§6)

### Step 5 — Auth (JWT)
1. `POST /api/auth/register` — hash password with bcrypt (cost ≥ 10), insert into `users`.
2. `POST /api/auth/login` — verify password, issue JWT with short expiry.
3. `auth.js` middleware — verifies `Authorization: Bearer <token>` on protected routes.

### Step 6 — Slot endpoints (public reads)
1. `GET /api/slots?date=YYYY-MM-DD` — cache-aside against Redis (`slot:{slotId}:available`), falling back to Postgres on a cache miss or Redis outage (fail-open per §10).
2. `GET /api/slots/:id` — same pattern for a single slot.
3. Add pagination (`?page=&limit=`) per §4/§8.

### Step 7 — Booking endpoint (the core concurrency requirement)
Implement exactly the atomic pattern from design §5 — no `SELECT` + separate `INSERT`:

```sql
BEGIN;
UPDATE gym_slots
SET booked_count = booked_count + 1
WHERE id = $1 AND booked_count < capacity
RETURNING booked_count;
-- 0 rows -> rollback, return 409 SLOT_FULL
-- 1 row  -> INSERT INTO bookings (...) VALUES (...)
COMMIT;
```
1. Wrap in a Postgres transaction using a single client from the pool.
2. On the unique-violation from `uq_active_booking_per_user_slot`, translate to `409 ALREADY_BOOKED`.
3. On success: invalidate the Redis cache key for that slot, then fire an async (non-blocking) write to `activity_logs` in Mongo.
4. Apply the rate-limiter middleware to this route (§6).

### Step 8 — Cancellation endpoint
Implement the symmetric atomic pair from §5:
```sql
BEGIN;
UPDATE bookings SET status='cancelled', cancelled_at=now()
WHERE id=$1 AND user_id=$2 AND status='confirmed';

UPDATE gym_slots SET booked_count = booked_count - 1
WHERE id=$3 AND booked_count > 0;
COMMIT;
```
- `403` if the requester isn't the booking owner.
- `409` if already cancelled.
- Invalidate the Redis cache key on success, same as booking.

### Step 9 — `GET /api/bookings/me`
Paginated list of the authenticated user's bookings, joined with slot info.

### Step 10 — Error handling middleware
Central Express error handler mapping every failure to the shape in §4:
```json
{ "error": { "code": "SLOT_FULL", "message": "..." } }
```
Cover the explicit failure paths from §9: Postgres down → `503`, Redis down → fall back to Postgres / fail-open on rate limiting, Mongo down → swallow + log warning (never blocks booking), invalid input → `400` at the validation layer.

### Step 11 — Input validation
Joi/Zod schemas for every request body and query param, applied before controllers run (blocks bad data from ever reaching business logic, per §7).

### Step 12 — Frontend (React)
Minimal, functional UI — no design polish required per the assignment tips:
1. Login/Register page → stores JWT (e.g. in memory or localStorage).
2. Slots page → lists slots with remaining capacity, "Book" button per slot.
3. My Bookings page → lists the user's bookings with a "Cancel" button.
4. A small `api/` layer wrapping fetch/axios calls to the endpoints in §4, attaching the JWT header for protected calls.

### Step 13 — Test the concurrency scenario
This is the most heavily weighted item in the assignment. Before recording the video:
1. Seed a slot with `booked_count = 9`, `capacity = 10` (1 spot left).
2. Fire 3 concurrent `POST /api/bookings` requests for that slot (e.g. with `Promise.all` in a small script, or a tool like `autocannon`/`k6`).
3. Confirm exactly 1 succeeds (`201`), the other 2 get `409 SLOT_FULL`, and `booked_count` never exceeds `10`.

### Step 14 — Local run instructions (for your submission README)
Document, in the repo's own `README.md`:
- Env vars needed (`DATABASE_URL`, `MONGO_URL`, `REDIS_URL`, `JWT_SECRET`, `PORT`)
- How to run migrations
- How to start backend (`npm run dev`) and frontend (`npm run dev`/`start`)
- Optional: a `docker-compose.yml` bringing up Postgres + Redis + Mongo together (mentioned as a "nice bonus" in the guidelines, not mandatory)

---

## 4. Full Development Flow (order of operations)

```
1. Scaffold repo (backend + frontend folders, env files)
        ↓
2. Postgres schema migration (users, gym_slots, bookings + constraints/indexes)
        ↓
3. Mongo collections wired (activity_logs, notification_history)
        ↓
4. Redis client wired (cache-aside + rate-limit keys)
        ↓
5. Auth: register/login + JWT middleware
        ↓
6. Slot read endpoints (with Redis cache-aside)
        ↓
7. Booking endpoint (atomic conditional UPDATE — the core concurrency fix)
        ↓
8. Cancellation endpoint (symmetric atomic UPDATE)
        ↓
9. Bookings-list endpoint (paginated)
        ↓
10. Central error handler + validation middleware wired to every route
        ↓
11. React frontend consuming the API contract
        ↓
12. Concurrency scenario test (1 spot, 3 simultaneous requests)
        ↓
13. Write repo README (setup/run instructions) + record video walkthrough
```

---

## 5. What's Deliberately Not Built

Per §10 of the design and the assignment's "no additional features" rule:
- No waitlist / notify-when-available
- No admin UI for creating/editing slots (slots are pre-seeded via migration/seed script)
- No configurable per-slot capacity (fixed at 10)
- No idempotency-key handling on booking (the unique-active-booking constraint is the stated safeguard)
