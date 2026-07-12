# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`, matching the naming convention used by `add_to_collection()`. To find every call site, I first checked the one place I already knew imported it, `routes/watchlist/watchlist.py`, and updated both the import statement and the function call there. To make sure that was the *only* call site, I ran a project-wide search rather than trusting my own memory of the codebase.
**How I verified:** Ran `grep -rn "save_to_watchlist" --include="*.py" .` from the project root — this searches every `.py` file in the repo, not just the ones I'd already touched — and confirmed zero remaining references to the old name (only `add_to_watchlist` matches now). Beyond the static search, I also ran the app locally against a seeded SQLite DB (one user, one film) and exercised the endpoint end-to-end with curl to confirm the renamed function still works at runtime, not just at the text level:

- `POST /watchlist/<user_id>/add` with a valid `film_id` → `201`, entry created correctly by `add_to_watchlist()`.
- `POST /watchlist/<user_id>/add` with no `film_id` → `400` as expected (unchanged validation in the route).

This confirms the rename didn't break the request path. Note: `GET /watchlist/<user_id>` and `POST` with a nonexistent `film_id` both currently 500 — pre-existing bugs unrelated to the rename (`WatchlistEntry` has no `film` relationship/backref defined on the `Film` model, and `FilmNotFoundError` isn't caught in the watchlist route), not introduced by this change.

## Comment 2 — Deduplication
**What I did:** Read `add_to_collection()` in `services/collection_service.py` to understand its dedup pattern: before inserting, it queries for an existing entry with the same `user_id`/`film_id`, and if one is found it raises a dedicated error (`AlreadyInCollectionError`) instead of inserting. I wrote the equivalent by hand in `add_to_watchlist()` (`services/watchlist_service.py`): added an `AlreadyInWatchlistError` exception class, then a `WatchlistEntry.query.filter_by(user_id=..., film_id=...).first()` check before creating the new entry, raising `AlreadyInWatchlistError` if a match exists. I did not ask AI to generate this check — I wrote it directly from reading the collection service's logic. I also updated `routes/watchlist/watchlist.py` to catch `FilmNotFoundError` (404) and `AlreadyInWatchlistError` (409) around the `add_to_watchlist()` call, matching how `routes/collection.py` handles the analogous errors for `add_to_collection()` — without this, the new exception would just surface as an unhandled 500.
**How I verified:** I verified the duplicate-detection logic actually prevents a duplicate row (not just that it raises) by running the app locally against a seeded SQLite DB and exercising it with curl, twice, with the same `film_id`:

- First `POST .../add` with `film_id: 1` → `201`, entry created.
- Second `POST .../add` with the same `film_id: 1` → `409` with `{"error": "Film '1' is already on this user's watchlist"}`, confirming the dedup check fires on the actual second insert attempt rather than in isolation.
- `POST .../add` with a nonexistent `film_id` → `404` (previously this was an unhandled 500 — see Comment 1's verification note; fixed here as part of adding the same error-handling pattern the collection route uses).
- `POST .../add` with a new, distinct `film_id` (`2`) → `201`, confirming the check only blocks true duplicates and the happy path for other films is unaffected.

## Comment 3 — Missing test
**What I did:** Used `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py` as the model for this test — it's the existing test that covers the same "missing film ID" case for the collection feature. I created `tests/test_watchlist.py` and copied that test's structure: the same `app` fixture (isolated Flask app + in-memory SQLite DB, created/torn down per test) and the same `sample_user` fixture. I didn't need the `sample_film` fixture, since this test deliberately never creates a real film — it exists to prove the *absence* of one is handled correctly. The new test, `test_add_to_watchlist_nonexistent_film_raises`, calls `add_to_watchlist()` with a `film_id` that doesn't exist and asserts it raises `FilmNotFoundError`, exactly mirroring the collection test's `pytest.raises(FilmNotFoundError)` assertion. One deliberate deviation from the model test: the collection version uses a fake UUID string for `fake_film_id`, but I used the integer `999999`, since `Film.id` in the current pre-refactor models is still an `Integer`, not a UUID (Comment 6 covers migrating this).
**How I verified:** Ran `pytest tests/test_watchlist.py -v` and confirmed the new test passed (`1 passed`). I also ran the full suite with `pytest -v` to make sure the new test file didn't break anything for the existing collection tests — all 5 tests passed (the 4 pre-existing collection tests plus the new watchlist test).

## Comment 4 — Default visibility
**My position:** `WatchlistEntry.public` (`models.py:80`) should default to `False`, not `True`. Making a watchlist entry public should be an explicit, opt-in action, not something a user is placed into automatically the moment they log a film they intend to watch.

**Reasoning:** README.md describes CineLog as "a community film tracking app" where "users log films they've watched, rate them, and build collections" — the community/discovery value comes from *watched* activity: ratings, reviews, completed collections, the record of taste a user has built up and chosen to put in front of others. A watchlist is a different kind of data: it's a to-do list of intent, not a record of accomplishment, and it's the one place in the app that's still forward-looking and unfinished. Defaulting it to public means the "community" surface gets flooded with half-formed intentions — films someone added on a whim, forgot about, or would rather revise before anyone sees the list — rather than the curated, watched-and-rated activity that's actually the reason other users would want to look at someone's profile. Defaulting to `False` means the entries that *do* end up visible are ones a user deliberately chose to surface, which is what makes a "public watchlist" a meaningful signal on a community platform instead of just default noise sitting on every profile. This is the same design choice Spotify makes with playlists: private by default, public only when a user decides a specific playlist is worth attaching their name to — the same logic applies here, because a watchlist, like a playlist, is curated-in-progress rather than a finished, ratable artifact like a logged film.

**Tradeoff acknowledged:** The alternative — `public=True` by default — optimizes for maximizing the *volume* of shareable content on the community surface from day one, which matters for a young platform that needs a critical mass of visible activity to feel alive and worth returning to. Defaulting to private trades that immediate density for quality: fewer public watchlists at launch, but the ones that are public are more likely to reflect something a user actually wants seen. Given that CineLog already generates community content through logged/rated films and collections — the "finished" activity — the watchlist doesn't need to carry the same discovery burden, so I think the tradeoff favors `False` here specifically because this platform doesn't have to rely on watchlists to bootstrap its community feed.

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