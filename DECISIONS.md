# DECISIONS.md

1. **Indexes.** Which indexes did you create, and which query or guarantee does each one serve? Which one did you consider and reject?
We created unique indexes on `employees.emp_code` (enforces unique employee codes and speeds lookups) and `attendance_logs.(emp_code, date)` (enforces R1's single record per day and accelerates monthly queries). We added compound indexes on `attendance_logs.(date: -1, emp_code: 1)` to eliminate in-memory sorting for `/attendance`, `attendance_logs.(emp_code, punch_in: -1)` to resolve punch-outs instantly, and `employees.joined_on` for headcount filtering. We considered and rejected indexing the `history` array because audit entries are append-only and never queried by key.

2. **Punch-in race.** Two identical punch-in requests arrive at the same instant. Walk through exactly what happens in your code.
Both requests compute the attendance date and call `insert_one`. MongoDB's compound unique index on `(emp_code, date)` atomically accepts the first write (returning 201 Created). The concurrent write immediately raises a `DuplicateKeyError`, which our exception handler intercepts and converts into a clean 409 Conflict response.

3. **Ties.** Two employees are tied on late minutes at the cutoff of `limit`. What does your leaderboard return, and why?
Using MongoDB's `$setWindowFields` with `$rank`, tied employees share the same rank (e.g., 1, 2, 2, 4). The `limit` cutoff applies *after* ranking (`rank <= limit`), sorted secondarily by `emp_code` ascending. Thus, both tied employees are returned, intentionally producing more than `limit` rows as required by the contract.

4. **Headcount.** How does your department summary make sure employees with zero logs are counted?
The pipeline starts on `employees` (matching `joined_on <= month_end`), grouping by department to calculate total headcount *before* performing a `$lookup` to `attendance_logs`. Employees without logs produce empty arrays, ensuring zero-log employees are fully counted.

5. **One thing you would change** if this had to serve 100x the data.
We would maintain pre-aggregated monthly summary documents updated asynchronously via MongoDB Change Streams, avoiding expensive real-time cross-collection joins on 10M+ documents.
