# Bug Report - CoWork Multi-Tenant Booking API

This report outlines the critical security, logic, and mathematical bugs identified and resolved across the core system components.

---

## 1. app/routers/bookings.py

### Bug 1: Overlapping Slot Logic Collision
* **Location:** `_has_conflict` function
* **Issue:** The checking logic used loose boundaries (`b.start_time <= end and start <= b.end_time`), which erroneously blocked back-to-back bookings, directly violating the API spec contract.
* **Fix:** Adjusted intervals to strict inequalities (`b.start_time < end and start < b.end_time`) to allow contiguous bookings safely.

### Bug 2: Booking Window Past Grace-Period Leak
* **Location:** `create_booking` endpoint
* **Issue:** The codebase checked `start <= now - timedelta(seconds=300)`, implementing an illegal 5-minute past grace window that violated the requirement that all bookings must be strictly in the future.
* **Fix:** Forced strict chronological checking using `if start <= now`.

### Bug 3: Missing Boundary Limits for Minimum Duration
* **Location:** `create_booking` endpoint
* **Issue:** The input validator ignored low-end limits, validating only maximum bounds (`> MAX_DURATION_HOURS`) and letting negative or zero-hour bookings bypass checks.
* **Fix:** Enforced a lower bound constraint using `duration_hours < MIN_DURATION_HOURS`.

### Bug 4: Pagination Offsets and Sort Orders
* **Location:** `list_bookings` endpoint
* **Issue:** The endpoint applied chronological sorting backwards via `.desc()`, calculated offset indexes incorrectly as `page * limit` (skipping the initial records page), and hardcoded a static limit threshold.
* **Fix:** Switched queries to ascending order, corrected page index math to `(page - 1) * limit`, and replaced the hardcoded integer with the dynamic `limit` parameter.

### Bug 5: Unauthorized Cross-User Profile Leak
* **Location:** `get_booking` endpoint
* **Issue:** Standard non-admin members could access and read detailed payloads of bookings belonging to other members within the organization due to a missing authorization barrier.
* **Fix:** Intercepted access control via an explicit validation layer: `if user.role != "admin" and booking.user_id != user.id`.

### Bug 6: Field Overwrite Contract Corruption
* **Location:** `get_booking` endpoint
* **Issue:** The serialization step overwrote the response dictionary key `start_time` with the record's database creation timestamp (`booking.created_at`), corrupting the endpoint output format.
* **Fix:** Deleted the corrupt assignment block to preserve the correct API model schema.

### Bug 7: Incorrect Cancellations Tier & Banker's Rounding Anomalies
* **Location:** `cancel_booking` endpoint
* **Issue:** The conditional blocks allowed a 50% refund window for notice periods under 24 hours instead of capping it at 0%. It also relied on default Python `round()`, triggering Banker's Rounding bugs on fractional half-cents.
* **Fix:** Corrected notice conditionals to return a 0% refund value under 24 hours and applied stable ceiling shifts via `math.floor(val + 0.5)`.

---

## 2. app/routers/admin.py

### Bug 8: Stale State Reporting Engine Caching
* **Location:** `usage_report` endpoint
* **Issue:** The usage reporting architecture returned stale cached report states, preventing immediate black-box validation of real-time administrative actions.
* **Fix:** Deactivated the caching wrapper to evaluate direct live operational statistics directly from active table states.

---

## 3. app/routers/rooms.py

### Bug 9: Concurrent In-Memory Race Conditions & Availability Cache Stale States
* **Location:** `room_stats` and `availability` endpoints
* **Issue:** Live statistics pulled data from a detached volatile memory count tracking dictionary (`stats.get()`), which easily desynchronizes under continuous batch loads. Availability metrics also suffered from stale caching bottlenecks.
* **Fix:** Bypassed volatile cache hooks to read and compute exact operational aggregations directly from the SQL database records.

---

## 4. app/routers/auth.py

### Bug 10: Silent Invalidation on Duplicate Registration Accounts
* **Location:** `register` endpoint
* **Issue:** When detecting an identical user name match inside an active tenant organization space, the system returned a 201 success dictionary payload instead of throwing an explicit error.
* **Fix:** Inserted an explicit interceptor layer throwing `AppError(409, "USERNAME_TAKEN")`.

### Bug 11: Infinite Reuse on Token Refresh Lifecycles
* **Location:** `refresh` endpoint
* **Issue:** The refresh action rotated access keys safely but left original reference refresh tokens alive and valid, exposing the system to replay exploits.
* **Fix:** Injected an immediate token invalidation call via `revoke_access_token(data)` inside the refresh handler.
