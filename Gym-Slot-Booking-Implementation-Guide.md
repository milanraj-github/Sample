# 🏋️ Gym Slot Booking — Complete Implementation Guide
### From Design to a Working App, Explained Step by Step in Simple Words

---

## Part 1: What Are We Actually Building? (In Plain Words)

Imagine a gym that has time slots — like "6 AM–7 AM", "7 AM–8 AM" — and each slot can only hold **10 people**. People log in, look at the slots, click "Book" if there's space, and can cancel later if they change their mind.

The **hard problem** isn't the booking itself — it's this: what if **3 people click "Book" on the same slot at the exact same second**, and there's only **1 spot left**? A badly built system might let all 3 in (now you have 11/10 people — overbooked). Our whole job is to make sure that **never happens**, no matter how much traffic hits the system at once.

Everything else (login, viewing slots, cancelling) is normal CRUD work. This concurrency problem is the heart of the project.

---

## Part 2: The Full Pipeline — How a Request Travels Through the System

Think of this as the journey of a single "Book this slot" click, from the user's browser all the way to the database and back.

```
 USER CLICKS "BOOK 6 AM SLOT"
        │
        ▼
 ┌─────────────────┐
 │  1. React App    │  Sends a POST request to /api/bookings
 │  (Frontend)      │  with the slot ID + JWT token in the header
 └────────┬─────────┘
          ▼
 ┌─────────────────┐
 │ 2. Express       │  The request lands on the Node.js server
 │    Server        │
 └────────┬─────────┘
          ▼
 ┌─────────────────┐
 │ 3. JWT Auth      │  "Is this a logged-in, valid user?"
 │    Middleware    │  No token / bad token → reject with 401
 └────────┬─────────┘
          ▼
 ┌─────────────────┐
 │ 4. Validation    │  "Is the request body shaped correctly?"
 │    Middleware    │  e.g. is slotId a valid UUID? → else 400
 └────────┬─────────┘
          ▼
 ┌─────────────────┐
 │ 5. Rate Limiter  │  "Has this user spammed the Book button?"
 │    (Redis)       │  Too many attempts/minute → 429
 └────────┬─────────┘
          ▼
 ┌─────────────────┐
 │ 6. Booking       │  The actual business logic lives here
 │    Controller    │
 └────────┬─────────┘
          ▼
 ┌───────────────────────────────────────┐
 │ 7. ONE ATOMIC SQL STATEMENT (Postgres) │  This is the safety-critical step —
 │    UPDATE gym_slots                    │  see Part 5 for why this is the
 │    SET booked_count = booked_count+1   │  entire solution to the race
 │    WHERE booked_count < capacity       │  condition problem.
 └────────┬────────────────────────────────┘
          ▼
   ┌──────┴──────┐
   │ Did it       │
   │ succeed?     │
   └──┬───────┬──┘
      │ NO    │ YES
      ▼       ▼
  409 SLOT_FULL   INSERT a row into `bookings` table
                          │
                          ▼
              ┌─────────────────────────┐
              │ 8. Invalidate Redis      │  Delete the cached "available
              │    cache key             │  seats" number so it's fresh
              └────────┬─────────────────┘
                       ▼
              ┌─────────────────────────┐
              │ 9. Fire-and-forget log   │  Write "user X booked slot Y"
              │    to MongoDB (async)    │  to Mongo — doesn't block or
              └────────┬─────────────────┘  fail the booking if Mongo is down
                       ▼
              ┌─────────────────────────┐
              │ 10. Response sent back   │  201 Created + booking details
              │     to React app         │
              └─────────────────────────┘
```

**In one sentence:** *Browser → Express → Auth check → Validation → Rate limit → one atomic database UPDATE that either wins or loses the last seat → cache cleared → audit log written in the background → response sent.*

The **viewing slots** pipeline is simpler and doesn't touch write-locks at all:

```
Browser → GET /api/slots → check Redis cache first
   → cache HIT: return cached numbers instantly
   → cache MISS: read from Postgres, save to Redis (10s TTL), return
```

---

## Part 3: Project Setup — Step by Step

### Step 1: Install prerequisites on your machine
- Node.js (v18+)
- PostgreSQL (v14+)
- MongoDB (v6+)
- Redis (v7+)
- (Optional but recommended) Docker + Docker Compose, so you can spin up Postgres/Mongo/Redis with one command instead of installing each separately

### Step 2: Create the project folders
```bash
mkdir gym-booking-system
cd gym-booking-system
mkdir backend frontend
```

### Step 3: Suggested backend folder structure
```
backend/
├── src/
│   ├── config/
│   │   ├── db.js          # Postgres connection pool
│   │   ├── mongo.js        # Mongo connection
│   │   └── redis.js        # Redis client
│   ├── middleware/
│   │   ├── auth.js         # JWT verification
│   │   ├── validate.js     # Joi/Zod request validation
│   │   └── rateLimiter.js  # Redis-backed rate limiting
│   ├── controllers/
│   │   ├── authController.js
│   │   ├── slotController.js
│   │   └── bookingController.js
│   ├── routes/
│   │   ├── authRoutes.js
│   │   ├── slotRoutes.js
│   │   └── bookingRoutes.js
│   ├── models/             # SQL queries live here (no heavy ORM required)
│   ├── utils/
│   │   └── errorHandler.js
│   └── app.js
├── migrations/
│   └── 001_init.sql        # the schema from Part 4
├── .env.example
└── package.json
```

### Step 4: Initialize the backend
```bash
cd backend
npm init -y
npm install express pg mongodb redis jsonwebtoken bcrypt joi dotenv cors helmet express-rate-limit
npm install -D nodemon
```

### Step 5: Set up environment variables (`.env`)
```env
PORT=5000
DATABASE_URL=postgres://user:password@localhost:5432/gymbooking
MONGO_URL=mongodb://localhost:27017/gymbooking
REDIS_URL=redis://localhost:6379
JWT_SECRET=your_super_secret_key
JWT_EXPIRES_IN=1h
```

---

## Part 4: Setting Up the Databases

### Step 1: Create the PostgreSQL database and run the schema
```bash
createdb gymbooking
psql gymbooking -f migrations/001_init.sql
```
The schema itself (users, gym_slots, bookings tables with the unique index that prevents double-booking, and the `CHECK` constraint that mathematically prevents overbooking) is exactly what's already defined in your design doc — that SQL is production-ready, just run it as-is.

### Step 2: Seed some gym slots for testing
```sql
INSERT INTO gym_slots (slot_date, start_time, end_time, capacity)
VALUES
  (CURRENT_DATE, '06:00', '07:00', 10),
  (CURRENT_DATE, '07:00', '08:00', 10),
  (CURRENT_DATE, '08:00', '09:00', 10);
```

### Step 3: MongoDB needs no schema
MongoDB is schema-less, so there's nothing to "create" — the `activity_logs` and `notification_history` collections are simply created automatically the first time you insert a document into them.

### Step 4: Redis needs no setup either
Just make sure the Redis server is running; keys are created on the fly by your code.

---

## Part 5: The Core Logic — Solving the Race Condition (Step by Step)

This is the part worth understanding slowly, because it's the whole point of the project.

**The wrong way (don't do this):**
```js
// ❌ BROKEN — two separate steps = race condition window
const slot = await db.query('SELECT booked_count, capacity FROM gym_slots WHERE id=$1', [slotId]);
if (slot.booked_count < slot.capacity) {
  await db.query('UPDATE gym_slots SET booked_count = booked_count + 1 WHERE id=$1', [slotId]);
  // Between the SELECT and this UPDATE, another request could have
  // read the SAME "9/10" and also decided there's room → overbooking.
}
```

**The right way (what this project uses):**
```js
// bookingController.js
async function createBooking(req, res) {
  const userId = req.user.id;       // set by auth middleware
  const { slotId } = req.body;
  const client = await pool.connect();

  try {
    await client.query('BEGIN');

    // Step 1: ONE atomic statement does the check AND the increment together.
    // Postgres locks this row for the instant it takes to run this line,
    // so two concurrent requests can NEVER both succeed on the last seat.
    const result = await client.query(
      `UPDATE gym_slots
       SET booked_count = booked_count + 1
       WHERE id = $1 AND booked_count < capacity
       RETURNING booked_count`,
      [slotId]
    );

    if (result.rowCount === 0) {
      await client.query('ROLLBACK');
      return res.status(409).json({ error: { code: 'SLOT_FULL', message: 'This slot has no remaining capacity.' } });
    }

    // Step 2: only reached if we "won" a seat — now record who booked it.
    const booking = await client.query(
      `INSERT INTO bookings (user_id, slot_id, status)
       VALUES ($1, $2, 'confirmed') RETURNING id, status`,
      [userId, slotId]
    );

    await client.query('COMMIT');

    // Step 3: clear the cached availability number so the next GET is fresh
    await redisClient.del(`slot:${slotId}:available`);

    // Step 4: fire-and-forget audit log — never blocks or fails the booking
    mongoLogsCollection.insertOne({
      userId, slotId, action: 'booking_created', timestamp: new Date()
    }).catch(err => console.warn('Mongo log failed (non-critical):', err));

    return res.status(201).json({ bookingId: booking.rows[0].id, slotId, status: 'confirmed' });

  } catch (err) {
    await client.query('ROLLBACK');
    if (err.code === '23505') { // Postgres unique-violation error code
      return res.status(409).json({ error: { code: 'ALREADY_BOOKED', message: 'You already booked this slot.' } });
    }
    return res.status(500).json({ error: { code: 'INTERNAL_ERROR', message: 'Something went wrong.' } });
  } finally {
    client.release();
  }
}
```

**Why this works, in plain words:** the database itself acts as the referee. When two requests race to update the same row, Postgres forces them to happen one after another (never truly simultaneously) at the row level. The one that runs first sees `booked_count < capacity` as true and wins the seat. The one that runs a millisecond later sees the *already-updated* count, finds the condition false, and its `UPDATE` simply affects 0 rows — no seat taken, no crash, just a clean "sorry, full" response.

**Cancellation follows the same atomic pattern** — cancel the booking row and decrement the counter inside one transaction.

---

## Part 6: Building Each Piece — Step by Step

### Step 1: Auth (Register/Login)
- `POST /api/auth/register`: validate input → hash password with bcrypt → insert into `users` table
- `POST /api/auth/login`: verify email exists → compare password with bcrypt → sign a JWT with `jsonwebtoken` → return the token

### Step 2: Auth middleware
```js
function authMiddleware(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: { code: 'UNAUTHENTICATED' } });
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch {
    return res.status(401).json({ error: { code: 'INVALID_TOKEN' } });
  }
}
```

### Step 3: Viewing slots (with caching)
```js
async function getSlots(req, res) {
  const { date } = req.query;
  const cacheKey = `slots:${date}`;
  const cached = await redisClient.get(cacheKey);
  if (cached) return res.json(JSON.parse(cached));

  const { rows } = await pool.query(
    `SELECT id, start_time, end_time, capacity, (capacity - booked_count) AS available
     FROM gym_slots WHERE slot_date = $1 ORDER BY start_time`,
    [date]
  );
  await redisClient.set(cacheKey, JSON.stringify(rows), { EX: 10 }); // 10s TTL
  res.json(rows);
}
```

### Step 4: Rate limiting middleware (Redis)
```js
async function bookingRateLimiter(req, res, next) {
  const key = `ratelimit:booking:${req.user.id}`;
  const attempts = await redisClient.incr(key);
  if (attempts === 1) await redisClient.expire(key, 60); // 1-minute window
  if (attempts > 5) {
    return res.status(429).json({ error: { code: 'RATE_LIMITED', message: 'Too many booking attempts, slow down.' } });
  }
  next();
}
```

### Step 5: Wiring up routes
```js
// bookingRoutes.js
router.post('/bookings', authMiddleware, validateBookingInput, bookingRateLimiter, createBooking);
router.delete('/bookings/:id', authMiddleware, cancelBooking);
router.get('/bookings/me', authMiddleware, getMyBookings);
```

### Step 6: Central error handler (so no raw stack traces ever leak)
```js
app.use((err, req, res, next) => {
  console.error(err);
  res.status(500).json({ error: { code: 'INTERNAL_ERROR', message: 'Something went wrong.' } });
});
```

### Step 7: Frontend (React) — minimal flow
1. `LoginPage` → calls `/api/auth/login` → stores JWT in memory/localStorage
2. `SlotsPage` → calls `/api/slots?date=...` on load → renders each slot with an "available" count and a **Book** button, disabled if `available === 0`
3. Clicking **Book** → calls `POST /api/bookings` with the JWT in the `Authorization` header → on `409 SLOT_FULL`, show a toast "Sorry, just filled up" and re-fetch slots
4. `MyBookingsPage` → calls `/api/bookings/me` → each row has a **Cancel** button calling `DELETE /api/bookings/:id`

---

## Part 7: Testing the Concurrency Fix (The Most Important Test)

This is how you *prove* the system works, not just assume it does.

```js
// concurrencyTest.js — fire 15 simultaneous booking requests at a slot with 10 seats left
const axios = require('axios');

async function testConcurrency(slotId, token) {
  const requests = Array.from({ length: 15 }, () =>
    axios.post('http://localhost:5000/api/bookings',
      { slotId },
      { headers: { Authorization: `Bearer ${token}` } }
    ).then(r => r.status).catch(e => e.response.status)
  );

  const results = await Promise.all(requests);
  const successes = results.filter(s => s === 201).length;
  const fulls = results.filter(s => s === 409).length;

  console.log(`✅ Successful bookings: ${successes} (should be exactly 10)`);
  console.log(`🚫 Rejected as full: ${fulls} (should be exactly 5)`);
}
```
If you run this and get **exactly 10 successes and 5 rejections, every single time** — no matter how many times you re-run it — your atomic `UPDATE` is working correctly.

---

## Part 8: Running Everything Locally

**Option A — Docker Compose (recommended, one command):**
```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: gymbooking
      POSTGRES_PASSWORD: password
    ports: ["5432:5432"]
  mongo:
    image: mongo:7
    ports: ["27017:27017"]
  redis:
    image: redis:7
    ports: ["6379:6379"]
```
```bash
docker compose up -d
```

**Option B — everything installed manually:**
```bash
# terminal 1
pg_ctl start        # or your OS's postgres service
# terminal 2
mongod
# terminal 3
redis-server
# terminal 4
cd backend && npm run dev
# terminal 5
cd frontend && npm start
```

---

## Part 9: Deployment Path (When You're Ready to Go Live)

1. **Backend** → deploy to a platform like Render/Railway/AWS EC2/ECS; set env vars there (never commit `.env`)
2. **PostgreSQL** → managed service (RDS, Supabase, Neon, Railway Postgres) for backups + connection pooling (PgBouncer)
3. **MongoDB** → MongoDB Atlas (managed, free tier available)
4. **Redis** → managed Redis (Upstash, Redis Cloud, or AWS ElastiCache)
5. **Frontend** → static build (`npm run build`) deployed to Vercel/Netlify
6. **HTTPS** → automatic on most of the above platforms; otherwise put a reverse proxy (Nginx) in front with a TLS cert (Let's Encrypt)
7. Set `NODE_ENV=production`, tighten CORS to your real frontend domain, and double check rate limits are active before opening it to real traffic

---

## Part 10: Quick Recap — The Whole Project in 5 Lines

1. **React** frontend shows slots and lets users book/cancel.
2. **Express** API authenticates (JWT), validates input, and rate-limits requests.
3. **PostgreSQL** is the single source of truth — one atomic `UPDATE ... WHERE booked_count < capacity` statement is what makes overbooking mathematically impossible, even under heavy concurrent load.
4. **Redis** only speeds up reads (caching) and controls abuse (rate-limiting) — it never decides correctness.
5. **MongoDB** silently logs activity in the background — it can go down without ever affecting a booking.

That's the entire system, start to finish.
