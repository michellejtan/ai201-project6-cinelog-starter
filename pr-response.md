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
* `CONTRIBUTING.md` states that any new service function needs a happy-path test, a duplicate/conflict test, and a nonexistent-ID test — `test_add_to_watchlist_nonexistent_film_raises` only covered the third of those. Once the dedup check was added in Comment 2, I added `test_add_to_watchlist_creates_entry` (happy path) and `test_add_to_watchlist_duplicate_raises` (asserts `AlreadyInWatchlistError` and that only one entry exists), mirroring `test_add_to_collection_creates_entry` and `test_add_to_collection_duplicate_raises` in `test_collection.py`.

**How I verified:**

* Ran `pytest tests/test_watchlist.py -v` to confirm the new tests passed.
* Ran `pytest tests/ -v` to ensure the complete test suite continued to pass — all 8 tests pass (4 in `test_collection.py`, 4 in `test_watchlist.py`).


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
**What conflicted:** While this PR was open, `refactor: migrate film IDs from integer to UUID` merged to `main`, changing `Film.id` (and every FK referencing it) from `db.Integer` to `db.String(36)`. I rebased `feature/watchlist` onto `main` with `git fetch origin && git rebase origin/main`. `models.py` conflicted directly: my branch still had the pre-refactor `WatchlistEntry` class (`user_id`/`film_id` as `db.Integer`-backed FKs) sitting alongside main's new UUID-based `Film`/`CollectionEntry`. Resolving the conflict by just taking main's side of the file silently dropped the `WatchlistEntry` class entirely, since it never existed on `main` — there was nothing on main's side for git to keep it against.

**How I resolved it:** I re-added `WatchlistEntry` to `models.py` on top of main's post-refactor schema, updating `id`, `user_id`, and `film_id` to `db.String(36)` to match the new UUID convention (previously `film_id` was `db.Integer`). I also updated the two remaining docstrings that still described `film_id` as an `int` — one in `add_to_watchlist()` in `services/watchlist_service.py`, and one in the request-body comment for `POST /watchlist/<user_id>/add` in `routes/watchlist/watchlist.py` — so nothing in the branch still implied integer film IDs.

**How I verified no conflict remains:** `git status` shows a clean rebase with no unresolved paths and no merge commits in `feature/watchlist`'s history (`git log --oneline --merges feature/watchlist` returns nothing, and `git merge-base feature/watchlist origin/main` equals `origin/main`'s tip). I then ran `pytest tests/ -v` — all 8 tests pass, including `test_get_watchlist_returns_newest_first`, which exercises `get_watchlist()` end-to-end and would fail immediately if the UUID plumbing were wrong (e.g. if `WatchlistEntry.film_id` still expected an integer, or if the `Film.watchlist_entries` relationship type mismatched).

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->

### What this PR does

Adds a watchlist feature to CineLog: users can save films they intend to watch (separate from their collection of films already watched). It adds a `WatchlistEntry` model, an `add_to_watchlist(user_id, film_id)` service function with duplicate and nonexistent-film handling, a `get_watchlist(user_id)` function that returns a user's saved films sorted by most-recently-added, and two endpoints — `GET /watchlist/<user_id>` and `POST /watchlist/<user_id>/add`. The feature follows the same patterns as the existing collection feature (`services/collection_service.py`), including its `verb_to_noun` naming, deduplication check, and `date_added.desc()` sort order.

### Design decisions

- **Default visibility (`public=True`)** — Watchlist entries default to public. CineLog's watchlist is meant for sharing what you plan to watch, not for private notes, so defaulting to visible optimizes for social discovery without adding an extra step for the common case. Full reasoning and the tradeoff (privacy-by-default, extra friction to share) in [Comment 4](#comment-4--default-visibility) above.
- **Sort order (newest first)** — `get_watchlist()` sorts by `date_added.desc()` rather than alphabetically, matching `get_collection()`'s existing behavior and surfacing what a user most recently decided to watch. Full reasoning, including the alphabetical-lookup tradeoff, in [Comment 5](#comment-5--sort-order) above.

### How to manually test

1. Start the app: `python app.py` (runs at `http://127.0.0.1:5000`).
2. Create a user and a film (or use existing seed data / the collection endpoints if they expose creation).
3. Add a film to the watchlist:
   ```
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_id>"}'
   ```
   Expect a `201` response with the new entry, including `"public": true`.
4. Fetch the watchlist:
   ```
   curl http://127.0.0.1:5000/watchlist/<user_id>
   ```
   Expect a list of films, most recently added first.
5. Repeat step 3 with the same `film_id` — expect the request to fail (`AlreadyInWatchlistError`) rather than creating a duplicate row.
6. Repeat step 3 with a `film_id` that doesn't exist — expect a `FilmNotFoundError` rather than a raw database error.
7. Run the automated test suite for full coverage: `pytest tests/ -v` (8 tests, all passing).

### git log --oneline
`git log --oneline main..feature/watchlist` — 14 commits, all conventional
format, no merge commits (a 15th commit, this one embedding the screenshot,
was added immediately afterward and isn't pictured):

![git log --oneline showing 14 conventional commits on feature/watchlist](assets/git-log-screenshot.png)

**Note for grading:** an unscoped `git log --oneline` on this branch also surfaces
`bbe206c Merge pull request #2 from ascherj/chore/add-gitignore`. That commit is
not part of this branch's own history — `git merge-base main feature/watchlist`
returns `bbe206c`, meaning it's the shared ancestor where `feature/watchlist` was
rebased onto `main`, merged into `main` by another contributor's PR before this
rebase. It predates and is external to every commit in this PR. The scoped log
above (`main..feature/watchlist`) is what GitHub's PR "Commits" tab shows, and
confirms this branch itself has a fully linear history with zero merge commits.