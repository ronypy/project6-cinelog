# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Filled in at the end -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` naming convention (`add_to_collection()`, `remove_from_collection()`, `get_collection()`). Updated the docstring's first line to match, updated the import and call site in `routes/watchlist/watchlist.py`, and left `WatchlistEntry`/behavior unchanged.
**How I verified:** Ran `grep -rn "save_to_watchlist\|add_to_watchlist" --include="*.py" .` before and after the change to confirm there was exactly one call site (`routes/watchlist/watchlist.py`) and that no reference to the old name remained afterward. Also ran `pytest tests/ -v` to confirm the existing suite still passes.

## Comment 2 — Deduplication
**What I did:** Added a duplicate check to `add_to_watchlist()` that mirrors `add_to_collection()`: after confirming the film exists, query for an existing `WatchlistEntry` with the same `user_id`/`film_id` and raise a new `AlreadyInWatchlistError` (defined in `watchlist_service.py`, parallel to `AlreadyInCollectionError`) if one is found, before creating the entry. I also updated `routes/watchlist/watchlist.py` to catch `AlreadyInWatchlistError` and return 409, and to actually catch `FilmNotFoundError` and return 404 — it was already imported there but never used, so a nonexistent film previously would have raised an unhandled 500.
**How I verified:** Ran the full test suite (`pytest tests/ -v`) to confirm no regressions, and manually traced both the collection and watchlist add paths side by side to confirm the duplicate-check logic and exception shapes match.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with the same `app`, `sample_user`, and `sample_film` fixtures as `tests/test_collection.py`, and added `test_add_to_watchlist_nonexistent_film_raises`, the direct equivalent of `test_add_to_collection_nonexistent_film_raises` — it asserts that calling `add_to_watchlist()` with a film ID that doesn't exist raises `FilmNotFoundError`.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` to confirm the new test passes, then `pytest tests/ -v` to confirm the full suite (5 tests) passes with no regressions.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default for new `WatchlistEntry` rows.

**Reasoning:** CineLog's whole premise is social film tracking — collections and ratings are already visible with no privacy toggle at all, so the app's implicit contract with users is "your activity is public unless you say otherwise." Watchlists are also the part of the product with the most social value: seeing what a friend wants to watch is what makes "let's watch something together" or a recommendation feed possible. If watchlist entries defaulted to private, that entire use case would be silently dead for every entry a user forgets to flip to public, and CineLog doesn't yet have a mechanism (notification, onboarding prompt, etc.) to make users aware they need to opt in. Matching the existing "public by default" behavior of collections also keeps the two features consistent, so a user doesn't have to learn two different privacy models within the same app.

**Tradeoff acknowledged:** The real cost of `public=True` is that a user's watchlist can reveal what they intend to watch — which is sometimes more personal than what they've already watched (e.g., it can telegraph interest in a sensitive topic before the user has decided whether they're comfortable sharing that). A first-time user who doesn't read documentation could be surprised that their watchlist is visible by default. If CineLog's user base grows beyond people who already expect a social, Letterboxd-like experience, that surprise-exposure risk gets larger, and a privacy-by-default (`public=False`) model would be safer for the least-informed user at the cost of muting the social feature for anyone who doesn't actively opt in.

## Comment 5 — Sort order
**My position:** Switched `get_watchlist()` from alphabetical (`Film.title.asc()`) to `date_added` descending (newest first), matching @dev-lead's suggestion.

**Reasoning:** I agree with the date-added ordering, mainly for a reason beyond "the maintainer asked for it": `get_collection()` already sorts by `date_added` descending, and a user moving between their collection view and their watchlist view within the same session shouldn't have to remember that one is newest-first and the other is alphabetical. A watchlist also behaves more like a queue than a catalog — when someone opens it, the question they're usually answering is "what did I just add / what's next," not "let me browse alphabetically," so newest-first actually serves the common case better than A–Z.

**Engagement with reviewer's point:** Alphabetical was my original implementation, but I don't think it holds up against the consistency argument above — nothing in the current API (no search or filter params) makes alphabetical browsing specifically valuable, so there's no real use case being sacrificed by dropping it as the default. If a future need for alphabetical browsing does show up (e.g., a very long watchlist), I'd rather solve that with an explicit `?sort=` query param than by making the two features default to different orders.

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
