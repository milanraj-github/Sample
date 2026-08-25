# 🗄️ Database Setup Guide — PostgreSQL, MongoDB, Redis
### Download, Install, and Basic Usage (Beginner Friendly)

There are two ways to do this: **Docker (easiest, recommended)** or **installing each one manually** on your OS. Both are covered below.

---

## Option A: The Easy Way — Docker (Recommended)

Instead of installing 3 separate databases, Docker runs all of them in lightweight containers with one command.

### Step 1: Install Docker Desktop
- Go to https://www.docker.com/products/docker-desktop and download for your OS (Windows/Mac/Linux)
- Install it like any normal app, then open it once to make sure it's running (you'll see a whale icon in your taskbar/menu bar)

### Step 2: Create a `docker-compose.yml` file in your project folder
```yaml
services:
  postgres:
    image: postgres:16
    container_name: gym_postgres
    environment:
      POSTGRES_DB: gymbooking
      POSTGRES_USER: gymuser
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  mongo:
    image: mongo:7
    container_name: gym_mongo
    ports:
      - "27017:27017"
    volumes:
      - mongodata:/data/db

  redis:
    image: redis:7
    container_name: gym_redis
    ports:
      - "6379:6379"

volumes:
  pgdata:
  mongodata:
```

### Step 3: Start everything with one command
```bash
docker compose up -d
```
That's it — Postgres, MongoDB, and Redis are now all running in the background. Check they're up:
```bash
docker ps
```
To stop them later: `docker compose down` (add `-v` to also wipe the data).

This is genuinely the least painful path — no version conflicts, no "it works on my machine" problems, and you can delete everything cleanly when done.

---

## Option B: Installing Each Database Manually

### 🐘 PostgreSQL

**Download:**
- **Windows/Mac:** https://www.postgresql.org/download/ → pick your OS → run the installer (on Windows it also installs **pgAdmin**, a visual tool)
- **Mac (via Homebrew, simpler):**
  ```bash
  brew install postgresql@16
  brew services start postgresql@16
  ```
- **Linux (Ubuntu/Debian):**
  ```bash
  sudo apt update
  sudo apt install postgresql postgresql-contrib
  sudo systemctl start postgresql
  ```

**Basic usage:**
```bash
# Open the interactive shell as the default admin user
sudo -u postgres psql

# Inside psql — create a database and a user
CREATE DATABASE gymbooking;
CREATE USER gymuser WITH PASSWORD 'password';
GRANT ALL PRIVILEGES ON DATABASE gymbooking TO gymuser;

# Exit
\q

# Connect to your new database directly from terminal
psql -U gymuser -d gymbooking -h localhost

# Run a .sql file (like your schema) against it
psql -U gymuser -d gymbooking -f migrations/001_init.sql
```
**Handy psql commands once connected:** `\dt` (list tables), `\d tablename` (describe a table), `\du` (list users), `\q` (quit).

**GUI option (easier for beginners):** install **pgAdmin** (https://www.pgadmin.org/) or **TablePlus** — connect using `localhost`, port `5432`, your username/password, and you get a visual table editor instead of typing SQL by hand.

---

### 🍃 MongoDB

**Download:**
- Go to https://www.mongodb.com/try/download/community → pick your OS
- **Mac (Homebrew):**
  ```bash
  brew tap mongodb/brew
  brew install mongodb-community@7.0
  brew services start mongodb-community@7.0
  ```
- **Linux (Ubuntu):** follow https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/ (adds Mongo's official apt repo, then `sudo apt install mongodb-org`)
- **Windows:** the downloaded `.msi` installer sets it up as a Windows service automatically

**Basic usage:**
```bash
# Start the Mongo shell (connects to localhost:27017 by default)
mongosh

# Inside the shell
use gymbooking          # switch to (or create) a database
db.activity_logs.insertOne({ userId: "abc", action: "test" })
db.activity_logs.find()  # view documents
db.activity_logs.find().pretty()
exit
```
Unlike Postgres, you never "create" a database or collection explicitly — it's created automatically the first time you insert data into it.

**GUI option:** install **MongoDB Compass** (comes bundled with the installer, or download separately from the same page) — lets you browse collections and documents visually instead of using the shell.

---

### 🔴 Redis

**Download:**
- **Mac (Homebrew):**
  ```bash
  brew install redis
  brew services start redis
  ```
- **Linux (Ubuntu):**
  ```bash
  sudo apt update
  sudo apt install redis-server
  sudo systemctl start redis-server
  ```
- **Windows:** Redis doesn't officially support Windows — easiest path is **WSL2** (Windows Subsystem for Linux) and then follow the Linux steps above, or just use the Docker option instead.

**Basic usage:**
```bash
# Open the Redis command-line client
redis-cli

# Inside redis-cli
SET slot:123:available 7        # store a value
GET slot:123:available          # read it back
DEL slot:123:available          # delete a key
EXPIRE slot:123:available 10    # set it to auto-expire in 10 seconds
TTL slot:123:available          # check remaining time-to-live
INCR ratelimit:booking:user1    # increment a counter (used for rate limiting)
KEYS *                          # list all keys (fine for local dev, never in production)
exit
```

**GUI option:** **RedisInsight** (free, from Redis themselves at https://redis.io/insight/) gives you a visual browser for keys and values.

---

## Quick Reference — Connecting from Your Node.js App

```env
# .env — these work whether you used Docker or manual install (same default ports)
DATABASE_URL=postgres://gymuser:password@localhost:5432/gymbooking
MONGO_URL=mongodb://localhost:27017/gymbooking
REDIS_URL=redis://localhost:6379
```

```js
// config/db.js — Postgres
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
module.exports = pool;

// config/mongo.js — MongoDB
const { MongoClient } = require('mongodb');
const client = new MongoClient(process.env.MONGO_URL);
async function connectMongo() {
  await client.connect();
  return client.db(); // returns the gymbooking database
}
module.exports = connectMongo;

// config/redis.js — Redis
const { createClient } = require('redis');
const redisClient = createClient({ url: process.env.REDIS_URL });
redisClient.connect();
module.exports = redisClient;
```

---

## Which Should You Pick?

| You are... | Use |
|---|---|
| New to databases, just want it working fast | **Docker** (Option A) |
| Want to learn what's happening under the hood | **Manual install** (Option B) |
| On Windows and want Redis to work smoothly | **Docker**, or WSL2 |
| Deploying for real later anyway | Doesn't matter locally — production will use managed services (RDS, Atlas, Upstash) regardless |

If you're unsure, **go with Docker** — it's one command, easy to reset (`docker compose down -v` wipes everything clean), and matches what most real teams do for local development anyway.
