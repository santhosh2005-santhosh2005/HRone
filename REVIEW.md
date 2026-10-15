# REVIEW.md

List every defect you found in the starter `app/main.py` (helpers and endpoints).

| # | Where (function / line) | What is wrong | How you'd notice it (test, input, or symptom) | How you fixed it |
|---|---|---|---|---|
| 1 | `health()` (line 72-74) | Returns hardcoded 200 without checking if MongoDB is reachable. | Shut down MongoDB and hit `GET /health`; it still returns 200 instead of 503. | Added `db.command("ping")` in try/except block; raises 503 if unreachable. |
| 2 | `create_employee()` (line 79-84) | Find-then-insert race condition. Two concurrent calls with the same `emp_code` can both succeed. | Send parallel `POST /employees` with identical `emp_code`; both insert. | Created unique index on `employees.emp_code` at startup and caught `DuplicateKeyError` to return 409. |
| 3 | `create_employee()` (line 81-85) | `created_at` stored as naive local datetime and returned as ISO string instead of epoch ms integer. | Check response JSON; `created_at` is a string instead of epoch milliseconds. | Stored UTC datetime truncated to seconds and return integer epoch milliseconds. |
| 4 | `EmployeeIn` (lines 53-61) | Missing schema validations on `emp_code` format, email regex, and shift times. | Passing `emp_code="bad"` or matching `shift_start` and `shift_end` returns 201 instead of 422. | Added Pydantic field validators with patterns for `emp_code`, email, and shift differences. |
| 5 | `list_employees()` (line 93) | Pagination skip formula is `page * page_size` instead of `(page - 1) * page_size`. | Requesting `page=1` skips the first 20 records. | Changed skip calculation to `(page - 1) * page_size`. |
| 6 | `list_employees()` (line 94) | `count_documents({})` runs without filter query. | Filter by `department=Sales`; total shows full company headcount instead of department total. | Passed filter dict `q` into `count_documents(q)`. |
| 7 | `list_employees()` (line 95) | Query cursor has no sort order. | Response order is non-deterministic instead of sorted by `emp_code` asc. | Added `.sort("emp_code", 1)` to the cursor. |
| 8 | `punch_in()` (lines 104-105) | Find-then-insert race condition allows duplicate punches for the same employee and date. | Send concurrent punch-ins for same date; both return 201. | Added unique index on `attendance_logs.(emp_code, date)` and return 409 on `DuplicateKeyError`. |
| 9 | `punch_in()` (line 101) | Does not check if employee exists before reading `shift_start`, causing unhandled 500. | Call punch-in with non-existent `emp_code`; crashes with `NoneType` error instead of 404. | Added `if not emp: raise HTTPException(404, "Employee not found")`. |
| 10 | `punch_in()` (lines 102-103) | R1 overnight shift rule ignored; date is taken directly from local punch time. | Overnight worker (22:00-06:00) punching in at 01:00 gets next day's date instead of shift start date. | Implemented `compute_attendance_date` checking if punch falls before `shift_end` on overnight shifts. |
| 11 | `compute_late_minutes()` (lines 33-37) | Grace period logic fails on exact boundaries and mixes naive timezones. | Punching in at 09:40:00 (inside grace) vs 09:40:01 (10 min late) gives incorrect values. | Replaced with elapsed seconds calculation: `floor(elapsed / 60)` only when `elapsed > 600`. |
| 12 | `punch_in()` (line 119) | Returns internal Mongo `_id` as string `id`, which is not in the OpenAPI schema. | OpenAPI validator fails response due to unexpected `id` field. | Removed `id` from returned dictionary and returned `AttendanceRecord` shape. |
| 13 | `list_attendance()` (lines 144-147) | Loads all matching documents into Python list and sorts in memory. | Hits memory limits and high latency on 100k records. | Replaced with MongoDB native `.sort().skip().limit()` backed by compound indexes. |
| 14 | `list_attendance()` (line 145) | Sorts only by date descending, missing secondary sort on `emp_code`. | Multiple records on same date appear in random employee order. | Sorted by `[("date", -1), ("emp_code", 1)]`. |
| 15 | `list_attendance()` (lines 125-132) | Missing `date_from > date_to` validation. | Passing `date_from` after `date_to` returns 200 with empty list instead of 422. | Added validation check raising 422 when `date_from > date_to`. |
| 16 | `list_attendance()` (lines 148-150) | Returns raw BSON dates instead of epoch milliseconds. | Frontends receive Python datetime objects / ISO strings instead of int64 milliseconds. | Serialized `punch_in`, `punch_out`, and history timestamps to epoch milliseconds. |

## What was examined and decided was NOT a defect
* **`compute_work_hours` core approach**: Dividing difference in seconds by 3600 matches rule R4. The only required adjustment was using `Decimal` half-up rounding instead of standard float rounding.
* **Default shifts ("09:30", "18:30")**: These match the standard shift timing assumptions in the project spec and sample documents.
