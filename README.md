# Employee Attendance & Analytics API

A FastAPI and MongoDB backend service implementing employee attendance tracking (punch-in, punch-out, regularization audit trails) and real-time analytics aggregation pipelines compliant with strict business rules (R1–R10).

## Requirements
* Python 3.11+
* MongoDB 6.0+ (Tested on MongoDB 7.0)

## Quick Start

1. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

2. **Configure environment:**
   Set environment variables or use a local `.env`:
   ```bash
   MONGO_URI="mongodb://localhost:27017"
   MONGO_DB="attendance_db"
   ```

3. **Seed initial sample data (optional):**
   ```bash
   python sample_seed.py
   ```

4. **Run the API server:**
   ```bash
   uvicorn app.main:app --port 8000
   ```
   The service automatically initializes all unique and performance-critical indexes on startup.

## Core Features & Endpoints
* **Health Check**: `GET /health` (probes MongoDB responsiveness, returns 200 or 503).
* **Employee Management**: `POST /employees`, `GET /employees` (with unique `emp_code` enforcement and paginated sorting).
* **Attendance Punch Events**:
  * `POST /attendance/punch-in`: Race-safe punch-in with R1 IST/overnight shift handling and R2 late calculation.
  * `POST /attendance/punch-out`: Race-safe punch-out computing R4 work hours, R3 overtime ($\ge$ 30 min), and R5 half-day.
  * `GET /attendance`: High-performance indexed pagination with date range filtering.
  * `PATCH /attendance/{emp_code}/{date}`: Manual regularizations with optimistic locking and complete history audit trail.
* **Analytics Aggregations**:
  * `GET /analytics/employees/{emp_code}/monthly`: Monthly stats accounting for mid-month joiners (R7).
  * `GET /analytics/departments/summary`: Department-level summary counting zero-log employees (R9).
  * `GET /analytics/leaderboard/late`: Dense competition ranking (1, 2, 2, 4) with post-ranking limits.
  * `GET /analytics/departments/{department}/trend`: Gap-filled calendar daily series (`$densify`) with 7-day moving averages.
* **Query Verification**: `GET /admin/explain/{endpoint}` providing raw execution stats proving `IXSCAN` usage.

## Deliverables
* [`app/main.py`](app/main.py): Self-contained application code.
* [`requirements.txt`](requirements.txt): Pinned dependencies.
* [`REVIEW.md`](REVIEW.md): Code review audit of starter defects.
* [`DECISIONS.md`](DECISIONS.md): Architectural decisions and index strategy.
