# 🔗 URL Shortener: System Design with Flask, PostgreSQL & Redis

A containerized URL shortener built to explore real-world system design concepts: Base62 encoding, the cache-aside pattern, multi-service orchestration, and Redis-based rate limiting. The whole stack runs locally with a single Docker Compose command.

![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-3.1.3-000000?logo=flask&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-7-DC382D?logo=redis&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)

---

## 📌 Overview

Send the API a long URL and get back a short code. Visit the short code and you get redirected to the original URL. Behind that simple interface sits a design that mirrors how production shorteners work:

- **PostgreSQL** is the durable source of truth for every URL mapping.
- **Redis** sits in front as a cache, so repeat lookups never touch the database.
- **Base62 encoding** turns auto-incremented database IDs into compact, URL-safe codes.
- **Docker Compose** wires all three services together on a private network with health checks and ordered startup.

---

## 🏗️ Architecture

```
                    ┌──────────────────────────────────────────┐
                    │          Docker Compose Network          │
                    │                                          │
 ┌────────┐  HTTP   │  ┌──────────────┐       ┌────────────┐   │
 │ Client │ ──────▶ │  │  Flask API   │ ────▶ │   Redis 7  │   │
 │ (curl) │ ◀────── │  │  web :5000   │ ◀──── │   (cache)  │   │
 └────────┘         │  └──────┬───────┘       └────────────┘   │
                    │         │                                │
                    │         ▼                                │
                    │  ┌──────────────┐                        │
                    │  │ PostgreSQL 16│ ◀── postgres_data vol  │
                    │  │  (database)  │                        │
                    │  └──────────────┘                        │
                    └──────────────────────────────────────────┘
```

### Shortening flow (`POST /shorten`)
1. The API inserts the long URL into PostgreSQL and gets back the auto-incremented `id`.
2. The `id` is encoded in Base62 (e.g. `125` → `21`).
3. The `short_code → long_url` mapping is written to Redis.
4. The short URL is returned with `201 Created`.

### Redirect flow (`GET /<short_code>`)
1. The API checks Redis first. **Cache hit** → redirect immediately.
2. **Cache miss** → decode the short code back to an `id`, query PostgreSQL, repopulate Redis.
3. Respond with a `302 Found` and a `Location` header pointing to the original URL.

---

## ✨ Features

- **Base62 short codes** using `0-9`, `a-z`, `A-Z` for compact, collision-free IDs
- **Cache-aside pattern** with Redis in front of PostgreSQL
- **302 redirects** so every visit hits the server (keeps analytics and cache logic in play)
- **Health checks** on every service, with `depends_on` conditions controlling startup order
- **Persistent storage** through a named Docker volume
- **Redis rate limiting** using atomic `INCR` counters and TTL expiration to block abusive clients
- **Structured logs** showing `CACHE HIT` and `CACHE MISS` events for easy observability

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| API | Python 3.12, Flask 3.1.3 |
| Database | PostgreSQL 16 (Alpine), psycopg2-binary 2.9.12 |
| Cache / Rate Limiting | Redis 7 (Alpine), redis-py 5.3.1 |
| Orchestration | Docker, Docker Compose |
| Tooling | Cursor, Terminal / PowerShell, WSL 2 (Windows) |

---

## 📁 Project Structure

```
url-shortener/
├── docker-compose.yml      # web, db, and redis services + volume + health checks
└── app/
    ├── Dockerfile          # python:3.12-slim base image
    ├── requirements.txt    # Flask, redis, psycopg2-binary
    ├── init.sql            # creates the urls table
    └── app.py              # Flask app, Base62 logic, routes, rate limiter
```

### Database schema

```sql
CREATE TABLE IF NOT EXISTS urls (
    id          SERIAL PRIMARY KEY,
    long_url    TEXT NOT NULL,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 🚀 Getting Started

### Prerequisites
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (includes Docker Compose)
- Windows users: WSL 2 enabled

Verify your install:

```bash
docker --version
docker compose version
```

### Run it

```bash
git clone https://github.com/<your-username>/url-shortener.git
cd url-shortener

# Optional: validate the compose file
docker compose config

# Build and start all services in the background
docker compose up --build -d

# Confirm everything is healthy
docker compose ps
```

The API is now live at `http://localhost:5000`.

---

## 📡 API Reference

### `GET /health`
Returns the service status.

```bash
curl http://localhost:5000/health
```

### `POST /shorten`
Creates a short URL.

```bash
curl -X POST http://localhost:5000/shorten \
  -H "Content-Type: application/json" \
  -d '{"url": "https://www.example.com/some/very/long/path"}'
```

**Response (`201 Created`)**
```json
{
  "short_code": "1",
  "short_url": "http://localhost:5000/1",
  "long_url": "https://www.example.com/some/very/long/path"
}
```

### `GET /<short_code>`
Redirects to the original URL.

```bash
curl -I http://localhost:5000/1
```

**Response**
```
HTTP/1.1 302 FOUND
Location: https://www.example.com/some/very/long/path
```

### Rate limiting
Each client IP gets a fixed number of requests per time window. Requests over the limit receive `429 Too Many Requests` until the Redis key's TTL expires.

---

## 🔍 Seeing the Cache in Action

```bash
# 1. Create a link and visit it twice
curl -X POST http://localhost:5000/shorten -H "Content-Type: application/json" -d '{"url":"https://github.com"}'
curl -I http://localhost:5000/1
curl -I http://localhost:5000/1

# 2. Watch the logs for CACHE HIT
docker compose logs web

# 3. Wipe the cache
docker compose exec redis redis-cli FLUSHALL

# 4. Visit again: you'll see CACHE MISS, then the entry is repopulated
curl -I http://localhost:5000/1
docker compose logs web
```

---

## 🧠 Design Decisions

**Why Base62?** Sequential IDs encoded in Base62 are short, URL-safe, and guaranteed unique without hashing or collision checks. Seven characters cover over 3.5 trillion URLs.

**Why cache-aside?** Redirects are extremely read-heavy. Letting the app check Redis first and fall back to PostgreSQL keeps the hot path fast, while the database stays the single source of truth. If Redis is flushed or restarts, nothing is lost.

**Why 302 instead of 301?** A `301 Moved Permanently` gets cached by browsers, so repeat visits skip the server entirely. A `302 Found` sends every visit through the API, which keeps click tracking, link expiration, and cache behavior possible. The trade-off is slightly more server load.

**Why Redis for rate limiting?** `INCR` is atomic and `EXPIRE` handles window resets automatically, so the limiter stays correct even with concurrent requests and needs no cleanup job.

---

## 🛠️ Troubleshooting

| Problem | Fix |
|---|---|
| Port 5000 already in use (macOS) | Disable **AirPlay Receiver** in System Settings → General → AirDrop & Handoff, or change the host port in `docker-compose.yml` |
| Container exits immediately | Make sure `Dockerfile` and `app.py` were saved and are not empty |
| `/health` refuses connections | Run `docker compose ps` and check that `db` and `redis` report `healthy` |
| Database errors on startup | Check `docker compose logs db` to confirm `init.sql` ran |

---

## 🧹 Cleanup

```bash
docker compose stop          # pause (keeps data)
docker compose start         # resume

docker compose down -v       # remove containers and the database volume
docker system prune          # clear unused images and build cache
```

---

## 📚 What I Learned

- Designing a multi-service backend with a clear separation between storage and caching
- Implementing the cache-aside pattern and reasoning about cache hits, misses, and invalidation
- Using Docker Compose networking, named volumes, health checks, and startup dependencies
- Building a Redis rate limiter with atomic counters and TTL expiration
- Weighing redirect semantics (301 vs 302) against product requirements

---

## 🙌 Acknowledgments

Built as part of the [NextWork](https://nextwork.ai) **URL Shortener System Design** project.
