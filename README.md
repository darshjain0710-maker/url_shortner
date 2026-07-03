# High-Concurrency URL Shortener

A URL shortener built for high-throughput redirects, using FastAPI, PostgreSQL, and Redis. It caches active links in memory for fast lookups, logs click analytics in the background without slowing down redirects, and rate-limits abusive traffic.

## Features

- **URL Shortening:** Generates 7-character Base62 short codes using a cryptographically secure random generator.
- **High-Concurrency Caching:** Active URLs are cached in Redis with a 24-hour TTL, keeping redirect resolution to sub-millisecond speeds.
- **Asynchronous Analytics:** Visitor data (IP address, timestamp, user agent) is written to PostgreSQL via FastAPI background tasks, so the redirect response never waits on the database write.
- **Rate Limiting:** IP-based limiting (10 requests per 60 seconds), backed by Redis.
- **Frontend Dashboard:** A simple HTML/TailwindCSS interface for creating short links and checking analytics.

## Tech Stack

- **Backend:** Python 3.11, FastAPI, Uvicorn
- **Database:** PostgreSQL (`asyncpg`)
- **Cache / Rate Limiter:** Redis (`redis.asyncio`)
- **Load Testing:** Locust
- **Frontend:** HTML5, TailwindCSS (via CDN)

## Performance and Load Testing

The system is built to handle high-volume read traffic, so load testing focused on redirect performance under stress. Tests were run with Locust, disabling redirect-following (`allow_redirects=False`) to isolate the throughput of the caching and routing layers from the destination site's response time.

Over the course of the test, the service processed more than 1,600 requests with zero failures, sustaining throughput above 40 requests per second (peaking near 44 RPS) and a median response time of 260ms. These results point to the Redis caching layer and async database operations doing their job well under concurrent load.

## Prerequisites

- Python 3.11
- PostgreSQL server
- Redis server

## Installation and Setup

1. **Clone the repository and navigate into it.**
2. **Create and activate a virtual environment:**
   ```bash
   python3.11 -m venv venv
   source venv/bin/activate  # On Windows use: venv\Scripts\activate
   ```
3. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```
4. **Set up environment variables:**
   Create a `.env` file in the root directory with the following:
   ```env
   DATABASE_URL=postgresql://user:password@localhost:5432/dbname
   REDIS_URL=redis://localhost:6379/0
   ```
5. **Initialize the database:**
   Make sure your PostgreSQL database has these tables:
   - `urls` (columns: `original_url`, `short_code` [UNIQUE])
   - `clicks` (columns: `short_code`, `ip_address`, `user_agent`, `timestamp`)
6. **Start the application:**
   ```bash
   uvicorn main:app --host 0.0.0.0 --port 8000
   ```
   The dashboard will be live at `http://localhost:8000/`.
7. **Run the load test (optional):**
   With the FastAPI server running, launch Locust using the provided script:
   ```bash
   locust -f locustfile.py
   ```
   Open `http://localhost:8089` to configure spawn rate and user count, then start the test.

## API Endpoints

- `GET /`: Serves the frontend dashboard.
- `POST /shorten`: Accepts `{"url": "https://..."}` and returns the shortened URL.
- `GET /{short_code}`: Redirects to the original URL (resolved from Redis cache) and kicks off background analytics logging.
- `GET /stats/{short_code}`: Returns total click count and the 5 most recent visits for a given short code.
