# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used AI tools to understand existing CineLog patterns, especially how add_to_collection() handles duplicate entries and how tests are structured. I also used AI to review my reasoning for Comments 4 and 5 by asking what counterarguments a reviewer might raise. I used that feedback to make sure my responses acknowledged privacy and usability tradeoffs.

## Comment 1 — Rename
**What I did:**
* Renamed `save_to_watchlist()` to `add_to_watchlist()` to match the project's existing `verb_to_noun` naming convention.
* Updated the import and function call in `routes/watchlist/watchlist.py` so all references use the new function name.

**How I verified:**
* Searched the project to confirm there were no remaining references to `save_to_watchlist()`.
* Ran the test suite with `pytest tests/ -v` to verify the rename did not introduce any errors.

## Comment 2 — Deduplication
**What I did:**

* Added a duplicate check to `add_to_watchlist()` following the same pattern used by `add_to_collection()` in `services/collection_service.py`.
* If the user already has the same film in their watchlist, the function now raises the appropriate exception instead of creating a duplicate entry.

**How I verified:**
* Compared the implementation with the existing collection service to ensure it follows the same pattern.
* Ran `pytest tests/ -v` after making the change to confirm the existing tests still passed.

## Comment 3 — Missing test
**What I did:**

* Created `tests/test_watchlist.py`.
* Added a test that verifies `add_to_watchlist()` raises `FilmNotFoundError` when the requested film does not exist.
* Used `tests/test_collection.py` as the model so the new test follows the project's existing testing style.

**How I verified:**

* Ran `pytest tests/test_watchlist.py -v` to confirm the new test passed.
* Ran `pytest tests/ -v` to ensure the complete test suite continued to pass - all 5 tests pass.


## Comment 4 — Default visibility
**My position:** I decided to keep `public=True` as the default visibility for watchlist entries (`WatchlistEntry`).

**Reasoning:** CineLog is an app that helps people keep track of movies they want to watch and share their movie interests with others. Making watchlists public by default makes it easier for people to share movie ideas and find new movies from their friends. Since a watchlist is just a list of movies someone wants to see, not a private diary or personal review, I think public visibility is a reasonable default.

**Tradeoff acknowledged:**
I understand why watchlists could be private by default. Some users may not want others to see what they plan to watch, and a private default reduces the risk of accidentally sharing that information — it also matches the privacy-first approach many apps use. The downside is that it adds an extra step for users who want to share, and makes social discovery less immediate. If CineLog shifts toward being more of a personal tracking tool than a social one, a private default would be the better choice.

## Comment 5 — Sort order
**My position:** I agree with the reviewer. `get_watchlist()` now sorts by `WatchlistEntry.date_added.desc()` (newest first) instead of `Film.title.asc()` (alphabetical).

**Reasoning:** A watchlist represents films a user intends to watch, so the most recently added ones are the ones most likely to still be top of mind — that's what someone opening their watchlist wants to see first. This also matches `get_collection()`, which already sorts by `date_added.desc()`, so the two features behave consistently for anyone using both.

**Engagement with reviewer's point:** The reviewer's point was that recency, not alphabetical order, is what users care about when returning to their watchlist. I agree, and I'd add that consistency with the collection feature is a second reason to prefer it — an inconsistent default between the two would be a small but real surprise. Alphabetical order does make it easier to locate one specific title in a long list, which date-added sorting doesn't solve. Rather than pick a compromise now, I think that's worth flagging as a future enhancement (user-configurable sort) instead of trying to satisfy both cases with a single default today.

**Bug found while testing this fix:** `get_watchlist()` calls `entry.film.to_dict()`, but `Film` only declared a `db.relationship` (with `backref="film"`) for `CollectionEntry`, not `WatchlistEntry` — so `entry.film` didn't exist and `get_watchlist()` raised `AttributeError` any time it was called, regardless of sort order. No existing test caught this because none exercised `get_watchlist()` end-to-end. I added `watchlist_entries = db.relationship("WatchlistEntry", backref="film", lazy=True)` to `Film` in `models.py`, matching the existing `collection_entries` pattern, and added `test_get_watchlist_returns_newest_first` (mirroring `test_get_collection_returns_newest_first`) to cover both the sort order and this relationship.

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->