# Bug Report — CoWork

23 bugs found and fixed. For each: where it is, what was wrong and why it broke
behavior, and how it was fixed. Difficulty tag in parentheses. No API contract was
changed (paths, status codes, error codes, JSON field names, and JWT claims are
preserved); the deliberate `time.sleep()` calls were kept and races fixed with locks.

---

### Bug 1 — Access token lifetime (Medium)
- **Where:** `app/auth.py:50`, `create_access_token`
- **Bug & why:** `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)` made tokens last 900 *minutes* (15 hours). Rule 8 requires `exp − iat` = exactly 900 seconds.
- **Fix:** `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)` (15 min = 900 s).

### Bug 2 — Logout doesn't invalidate the token (Medium)
- **Where:** `app/auth.py:97`, `get_token_payload` (revocation stored at line 86)
- **Bug & why:** `revoke_access_token` stores the token's `jti`, but the check read `payload.get("sub") in _revoked_tokens`. A user id never equals a stored `jti`, so logged-out tokens kept working, violating rule 8.
- **Fix:** Check `payload.get("jti") in _revoked_tokens`.

### Bug 3 — Refresh token not single-use (Hard)
- **Where:** `app/routers/auth.py`, `refresh`; helper added in `app/auth.py`
- **Bug & why:** `/auth/refresh` issued new tokens but never invalidated the presented refresh token, so it could be replayed forever. Rule 8 requires refresh tokens to be single-use (reuse → 401).
- **Fix:** Added `consume_token_jti(jti)` in `auth.py` — atomically (under `_revocation_lock`) checks and marks the `jti` used, returning `False` on reuse. `refresh` calls it and raises 401 on reuse. The lock makes it hold under concurrent reuse.

### Bug 4 — Duplicate registration silently succeeds (Easy)
- **Where:** `app/routers/auth.py:37-43`, `register`
- **Bug & why:** Registering an existing username returned 201 with the existing user instead of an error. Rule 15 requires `409 USERNAME_TAKEN`.
- **Fix:** `raise AppError(409, "USERNAME_TAKEN", ...)`. Also wrapped the commit in `try/except IntegrityError` so a concurrent duplicate returns 409 (not a 500) and a lost org-create race joins the org as member.

### Bug 5 — Timezone offset dropped, not converted (Medium)
- **Where:** `app/timeutils.py:13`, `parse_input_datetime`
- **Bug & why:** For offset-aware input it did `dt.replace(tzinfo=None)`, discarding the offset (e.g. `20:00+06:00` stored as `20:00` UTC). Rule 1 requires conversion to UTC (`14:00`).
- **Fix:** `dt.astimezone(timezone.utc).replace(tzinfo=None)`.

### Bug 6 — Back-to-back bookings wrongly rejected (Medium)
- **Where:** `app/routers/bookings.py:50`, `_has_conflict`
- **Bug & why:** `b.start_time <= end and start <= b.end_time` used `<=`, so a booking starting exactly when another ends was flagged as a conflict. Rule 3 defines overlap with strict `<` and allows back-to-back.
- **Fix:** `b.start_time < end and start < b.end_time`.

### Bug 7 — Past-start grace window (Medium)
- **Where:** `app/routers/bookings.py:86`, `create_booking`
- **Bug & why:** `start <= now - timedelta(seconds=300)` allowed start times up to 5 minutes in the past. Rule 2 requires start strictly in the future, no grace.
- **Fix:** `start <= now`.

### Bug 8 — Zero/negative duration accepted (Medium)
- **Where:** `app/routers/bookings.py:93`, `create_booking`
- **Bug & why:** Only `> MAX_DURATION_HOURS` was checked; a 0-hour or negative (end ≤ start) duration passed the whole-number check and was booked. Rule 2 requires minimum 1 hour and end strictly after start.
- **Fix:** `if duration_hours < MIN_DURATION_HOURS or duration_hours > MAX_DURATION_HOURS:` (uses the existing `MIN_DURATION_HOURS = 1`).

### Bug 9 — Pagination order/offset/limit all wrong (Medium)
- **Where:** `app/routers/bookings.py:137-139`, `list_bookings`
- **Bug & why:** `order_by(start_time.desc())` (should be ascending), `offset(page * limit)` (page 1 skipped the first page), and hard-coded `.limit(10)` (ignored the `limit` param). Violates rule 11 (ascending, items `[(N−1)·L, N·L)`).
- **Fix:** `order_by(Booking.start_time.asc(), Booking.id.asc()).offset((page - 1) * limit).limit(limit)`.

### Bug 10 — Booking detail returns wrong start_time (Easy)
- **Where:** `app/routers/bookings.py:166`, `get_booking`
- **Bug & why:** `response["start_time"] = iso_utc(booking.created_at)` overwrote the correct start time with the creation timestamp.
- **Fix:** Deleted the line (`serialize_booking` already sets `start_time` correctly).

### Bug 11 — Refund tier thresholds wrong (Medium)
- **Where:** `app/routers/bookings.py:200-206`, `cancel_booking`
- **Bug & why:** `notice_hours > 48` (floored + strict) gave 50% for notice in `[48h,49h)`, and the final `else` returned `50` instead of `0`. Rule 6: ≥48h → 100%, 24–48h → 50%, <24h → 0%.
- **Fix:** `if notice >= timedelta(hours=48): 100 / elif notice >= timedelta(hours=24): 50 / else: 0`.

### Bug 12 — Refund rounding wrong and response ≠ RefundLog (Hard)
- **Where:** `app/services/refunds.py:15-17` (`log_refund`) and `app/routers/bookings.py:208` (`cancel_booking`)
- **Bug & why:** The response used `round()` (banker's rounding) while `log_refund` used float math + `int()` truncation, so they disagreed (e.g. rate 999 @ 50%: response 500, stored 499) and neither rounded half-up. Rule 6 requires half-up and the response amount to equal the stored RefundLog amount.
- **Fix:** `log_refund` uses integer half-up `(price_cents * percent + 50) // 100`; `cancel_booking` returns the created entry's `amount_cents`, so the two are identical by construction.

### Bug 13 — Usage report stale after a new booking (Medium)
- **Where:** `app/routers/bookings.py`, `create_booking` (post-commit)
- **Bug & why:** Creating a booking invalidated the availability cache but not the report cache, so a cached usage report kept serving stale counts. Rule 12 requires the report to reflect current state immediately.
- **Fix:** Added `cache.invalidate_report(user.org_id)` after commit.

### Bug 14 — Availability stale after cancel (Medium)
- **Where:** `app/routers/bookings.py`, `cancel_booking` (post-commit)
- **Bug & why:** Cancelling invalidated the report cache but not the availability cache, so a cancelled slot still showed as busy. Rule 13 requires availability to reflect current state immediately.
- **Fix:** Added `cache.invalidate_availability(booking.room_id, booking.start_time.date().isoformat())` after commit.

### Bug 15 — Export leaks cross-org bookings (Hard)
- **Where:** `app/services/export.py:48-50`, `generate_export` (via `fetch_bookings_raw`)
- **Bug & why:** With `include_all=true&room_id=<id>`, it called `fetch_bookings_raw(db, room_id)` with no org filter, so an admin could export another org's bookings. Rule 9: cross-org ids behave as non-existent.
- **Fix:** Route that branch through the org-scoped `_fetch_scoped(db, org_id, None, room_id)`.

### Bug 16 — Member can read another member's booking (Medium)
- **Where:** `app/routers/bookings.py`, `get_booking`
- **Bug & why:** `get_booking` filtered only by org, missing the ownership guard that `cancel_booking` has, so any member could read another member's booking. Rule 10: other member's booking id → 404.
- **Fix:** Added `if user.role != "admin" and booking.user_id != user.id: raise AppError(404, "BOOKING_NOT_FOUND", ...)`.

### Bug 17 — Double-booking under concurrency (Hard)
- **Where:** `app/routers/bookings.py`, `create_booking` / `_has_conflict`
- **Bug & why:** The conflict check and the insert/commit were not atomic (and `_pricing_warmup()` widened the gap), so concurrent requests for the same slot all passed the check and all committed. Rule 3 must hold under concurrency.
- **Fix:** Wrapped the conflict-check → quota → insert → commit section in a module-level `_booking_lock`.

### Bug 18 — Quota bypass under concurrency (Hard)
- **Where:** `app/routers/bookings.py`, `_check_quota`
- **Bug & why:** The quota count and the insert raced (widened by `_quota_audit()`), so a member could exceed 3 confirmed bookings in the 24h window. Rule 4 must hold under concurrency.
- **Fix:** Covered by the same `_booking_lock` critical section as Bug 17.

### Bug 19 — Rate limit bypass under concurrency (Hard)
- **Where:** `app/services/ratelimit.py`, `record_and_check`
- **Bug & why:** Read bucket → sleep → append → write back was not atomic, so concurrent requests read the same bucket and overwrote each other (lost updates); the counter never accumulated. Rule 5 must hold under concurrency.
- **Fix:** Guarded the read-trim-append with a `threading.Lock`, capturing the count inside the lock (sleep kept outside).

### Bug 20 — Duplicate reference codes under concurrency (Hard)
- **Where:** `app/services/reference.py`, `next_reference_code`
- **Bug & why:** Read counter → sleep → increment was not atomic, so concurrent creators got identical codes. Rule 7 requires uniqueness under concurrency.
- **Fix:** Wrapped the read+increment in a `threading.Lock` (sleep kept outside).

### Bug 21 — Room stats drift under concurrency (Hard)
- **Where:** `app/services/stats.py`, `record_create` / `record_cancel`
- **Bug & why:** Read stats → sleep → write back was not atomic, so concurrent updates overwrote each other and counts/revenue drifted below reality. Rule 14 requires stats consistent with the bookings.
- **Fix:** Guarded each read-modify-write with a shared `threading.Lock`.

### Bug 22 — Concurrent cancel creates multiple refunds (Hard)
- **Where:** `app/routers/bookings.py`, `cancel_booking`
- **Bug & why:** The `status == "cancelled"` check preceded `_settlement_pause()` and the commit, so concurrent cancels of the same booking all saw `confirmed`, all wrote a RefundLog, and all returned 200. Rule 6 requires exactly one RefundLog and holding under concurrency.
- **Fix:** Wrapped status-check → refund → commit in `_booking_lock` with `db.refresh(booking)` inside, so the losers see `cancelled` and get `409 ALREADY_CANCELLED`.

### Bug 23 — Deadlock between create and cancel (Hard)
- **Where:** `app/services/notifications.py:31-35`, `notify_cancelled`
- **Bug & why:** `notify_created` locked email→audit while `notify_cancelled` locked audit→email — opposite order (AB-BA). A concurrent create + cancel deadlocked and wedged the whole service. Rule 16 requires liveness.
- **Fix:** Acquire locks in the same order (email → audit) in both functions.

---

## Summary

| # | Bug | Difficulty | File |
|---|---|---|---|
| 1 | Access token lifetime 900 min not 900 s | Medium | app/auth.py |
| 2 | Logout checks `sub` but stores `jti` | Medium | app/auth.py |
| 3 | Refresh token not single-use | Hard | app/routers/auth.py |
| 4 | Duplicate register not 409 | Easy | app/routers/auth.py |
| 5 | Offset datetime not converted to UTC | Medium | app/timeutils.py |
| 6 | Back-to-back rejected (`<=`) | Medium | app/routers/bookings.py |
| 7 | Past-start grace window | Medium | app/routers/bookings.py |
| 8 | Zero/negative duration accepted | Medium | app/routers/bookings.py |
| 9 | Pagination order/offset/limit | Medium | app/routers/bookings.py |
| 10 | Detail start_time overwritten | Easy | app/routers/bookings.py |
| 11 | Refund tier thresholds | Medium | app/routers/bookings.py |
| 12 | Refund rounding + response ≠ RefundLog | Hard | app/routers/bookings.py, app/services/refunds.py |
| 13 | Usage report stale after create | Medium | app/routers/bookings.py |
| 14 | Availability stale after cancel | Medium | app/routers/bookings.py |
| 15 | Export leaks cross-org bookings | Hard | app/services/export.py |
| 16 | Member reads others' bookings | Medium | app/routers/bookings.py |
| 17 | Double-booking under concurrency | Hard | app/routers/bookings.py |
| 18 | Quota bypass under concurrency | Hard | app/routers/bookings.py |
| 19 | Rate limit bypass under concurrency | Hard | app/services/ratelimit.py |
| 20 | Duplicate reference codes | Hard | app/services/reference.py |
| 21 | Stats drift under concurrency | Hard | app/services/stats.py |
| 22 | Concurrent cancel → multiple refunds | Hard | app/routers/bookings.py |
| 23 | Create/cancel notification deadlock | Hard | app/services/notifications.py |
