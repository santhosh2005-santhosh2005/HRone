# Employee Attendance & Analytics API

FastAPI + MongoDB backend for employee attendance tracking and analytics. Handles punch events, shift calculations (including overnight shifts), manual regularizations with audit history, and aggregation-based reporting.

## Setup & Running

1. **Install requirements:**
   ```bash
   pip install -r requirements.txt
   ```

2. **Database configuration:**
   By default it connects to `mongodb://localhost:27017` on database `attendance_db`. You can override these using environment variables:
   ```bash
   MONGO_URI="mongodb://localhost:27017"
   MONGO_DB="attendance_db"
   ```

3. **Seed sample data (optional):**
   ```bash
   python sample_seed.py
   ```

4. **Start the API:**
   ```bash
   uvicorn app.main:app --port 8000
   ```
   All required unique and query indexes are automatically created at startup on MongoDB.

## Endpoints

* `GET /health`: Readiness probe checking MongoDB connection.
* `POST /employees`: Create employee (validates format, enforces unique `emp_code`).
* `GET /employees`: Paginated employee list sorted by `emp_code`.
* `POST /attendance/punch-in`: Record punch-in (handles overnight shifts and calculates late minutes).
* `POST /attendance/punch-out`: Close open punch-in (calculates work hours, overtime, and half-day status).
* `GET /attendance`: Paginated attendance history with date and employee filters.
* `PATCH /attendance/{emp_code}/{date}`: Manual correction endpoint; updates derived metrics and appends an audit record to `history`.
* `GET /analytics/employees/{emp_code}/monthly`: Monthly stats for an employee.
* `GET /analytics/departments/summary`: Department totals including zero-log employees.
* `GET /analytics/leaderboard/late`: Late arrivals leaderboard using competition ranking (`$rank`).
* `GET /analytics/departments/{department}/trend`: Calendar-gap-filled trend line using `$densify` with 7-day moving average.
* `GET /admin/explain/{endpoint}`: Returns raw MongoDB explain plan confirming `IXSCAN` usage.

## Files
* `app/main.py`: Complete API application.
* `REVIEW.md`: Audit of defects found in the starter code.
* `DECISIONS.md`: Design and architecture decisions.
* `requirements.txt`: Python package dependencies.
