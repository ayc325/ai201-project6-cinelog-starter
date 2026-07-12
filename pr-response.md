# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`, matching the naming convention used by `add_to_collection()`. Updated the one call site in `routes/watchlist/watchlist.py` (both the import and the function call).
**How I verified:** Ran `grep -rn "save_to_watchlist" --include="*.py" .` across the whole project to confirm no remaining references to the old name. Also ran the app locally against a seeded SQLite DB (one user, one film) and exercised the endpoint end-to-end with curl:

- `POST /watchlist/<user_id>/add` with a valid `film_id` → `201`, entry created correctly by `add_to_watchlist()`.
- `POST /watchlist/<user_id>/add` with no `film_id` → `400` as expected (unchanged validation in the route).

This confirms the rename didn't break the request path. Note: `GET /watchlist/<user_id>` and `POST` with a nonexistent `film_id` both currently 500 — pre-existing bugs unrelated to the rename (`WatchlistEntry` has no `film` relationship/backref defined on the `Film` model, and `FilmNotFoundError` isn't caught in the watchlist route), not introduced by this change.

## Comment 2 — Deduplication
**What I did:**
**How I verified:**

## Comment 3 — Missing test
**What I did:**
**How I verified:**

## Comment 4 — Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->