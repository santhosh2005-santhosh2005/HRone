# DECISIONS.md

1. **Indexes.** Which indexes did you create, and which query or guarantee does each one serve? Which one did you consider and reject?
I added unique indexes on `employees.emp_code` and `attendance_logs.(emp_code, date)` to enforce data uniqueness and prevent race conditions. For query speed, I indexed `(date: -1, emp_code: 1)` on logs to paginate `/attendance` without in-memory sorting, `(emp_code, punch_in: -1)` to find open punches quickly, and `employees.joined_on` for headcount filters. I considered indexing the `history` array but rejected it because it is an append-only audit log that is never queried by key.

2. **Punch-in race.** Two identical punch-in requests arrive at the same instant. Walk through exactly what happens in your code.
Both requests determine the date and run `insert_one`. The database unique index on `(emp_code, date)` allows the first write through (returning 201). The second write fails with `DuplicateKeyError`, which my code catches and returns as a 409 Conflict.

3. **Ties.** Two employees are tied on late minutes at the cutoff of `limit`. What does your leaderboard return, and why?
The pipeline uses `$rank` window ranking, so tied employees share the same rank and sort by `emp_code`. Because `limit` is applied after ranking (`rank <= limit`), both tied employees are included in the output, which can exceed `limit` items as expected.

4. **Headcount.** How does your department summary make sure employees with zero logs are counted?
The pipeline starts on `employees`, matching anyone who joined on or before the month end. It groups by department to establish base headcount before looking up logs, so employees with no records still get counted.

5. **One thing you would change** if this had to serve 100x the data.
At 10M+ rows, I would stop doing real-time cross-collection joins and instead maintain pre-computed monthly summary documents updated asynchronously via change streams.
