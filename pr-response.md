# PR Response Doc — CineLog Watchlist Feature

This document records how I addressed the six review comments left by the maintainer
(`@dev-lead`) on the watchlist PR, plus the two design decisions and the rebase.

## AI Usage

I used Cursor's AI assistant during this project in several ways:

1. **Codebase orientation:** Before reading the PR comments, I had the AI summarize
   `models.py`, `services/collection_service.py`, and `tests/test_collection.py` to
   understand naming conventions (`verb_to_noun`), deduplication patterns, and test
   fixture structure.
2. **Finding call sites:** After renaming `save_to_watchlist` to `add_to_watchlist`, I
   used project-wide search (`grep`) to confirm zero remaining references to the old name.
3. **Rebase recovery:** After rebasing onto `main`, the `WatchlistEntry` model was missing
   from `models.py`. I identified this via failing tests and restored the model with UUID
   `film_id` to match the post-refactor `main` branch.
4. **Stress-testing design arguments (Comments 4 & 5):** I asked the AI what counterarguments
   a reviewer might raise against defaulting to `public=True` and against date-added sort
   order. The AI noted privacy concerns for public defaults and alphabetical lookup benefits
   for large watchlists. I incorporated the privacy tradeoff explicitly in Comment 4 and
   addressed the alphabetical lookup point in Comment 5 by noting that date-added matches
   collection behavior and user expectations for "recently saved" items.

---

## Comment 1 — Rename
> `save_to_watchlist()` should follow the project's naming convention. Compare with
> `add_to_collection()` — the pattern here is `verb_to_noun`. Please rename to
> `add_to_watchlist()` and update all call sites.

**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`
so it matches the project's `verb_to_noun` service naming convention (the same pattern as
`add_to_collection()`, `remove_from_collection()`, `get_collection()`). I then updated the
single call site in `routes/watchlist/watchlist.py` — both the `import` line and the call
inside the `add_film` view.

**How I verified:**
Ran a project-wide search (`grep -rn "save_to_watchlist" --include="*.py"`) to confirm zero
references to the old name remained. I also imported the app and the renamed function
(`python -c "import app; app.create_app(); from services.watchlist_service import add_to_watchlist"`)
to confirm nothing broke at import time. The full test suite still passed after the change.

---

## Comment 2 — Deduplication
> What happens if a user calls this with a film that's already on their watchlist? The
> current implementation would add a duplicate entry. Please handle this case.

**What I did:**
Studied `add_to_collection()` in `services/collection_service.py`, which queries for an
existing entry with `filter_by(user_id=..., film_id=...)` and raises
`AlreadyInCollectionError` if one exists. I applied the same pattern to `add_to_watchlist()`:
added an `AlreadyInWatchlistError` exception class and a pre-insert check that raises it when
a duplicate is detected. I also updated `routes/watchlist/watchlist.py` to return HTTP 409
(Conflict) for duplicates, matching how the collection route handles `AlreadyInCollectionError`.

**How I verified:**
Ran `pytest tests/ -v` — all tests passed. Manually traced the logic: the function now checks
`db.session.get(Film, film_id)` first (raises `FilmNotFoundError`), then queries for an
existing `WatchlistEntry` (raises `AlreadyInWatchlistError`), and only then creates a new entry.
After the rebase, I also added a `UniqueConstraint` on `(user_id, film_id)` in the
`WatchlistEntry` model as a database-level safeguard, mirroring `CollectionEntry`.

---

## Comment 3 — Missing test
> Please add a test for the case where `film_id` doesn't exist in the database. Look at
> the existing tests in `test_collection.py` — the pattern is there.

**What I did:**
Created `tests/test_watchlist.py` modeled directly on `tests/test_collection.py`. I copied the
same fixture structure (`app`, `sample_user`, `sample_film`) and wrote
`test_add_to_watchlist_nonexistent_film_raises`, which mirrors
`test_add_to_collection_nonexistent_film_raises`: it uses a fake UUID
(`00000000-0000-0000-0000-000000000000`) and asserts that `add_to_watchlist()` raises
`FilmNotFoundError`.

**How I verified:**
Ran `pytest tests/test_watchlist.py -v` — the new test passed. Ran `pytest tests/ -v` — full
suite passed (5 tests).

---

## Comment 4 — Default visibility
> I notice watchlists default to `public=True`. We don't have a documented decision on
> default visibility for user lists. Before I can approve this, I need you to add a note
> to your PR description explaining your reasoning. I want to make sure we're being
> intentional here, not just inheriting a default.

**My position:**
Keep `public=True` as the default for new watchlist entries.

**Reasoning:**
CineLog is described as a community film-tracking app — users log films, rate them, and
build collections. That framing implies social discovery is a core value, not an optional
add-on. A watchlist is inherently forward-looking ("films I plan to watch"), which makes it
useful social signal: friends and community members can see what someone is excited about,
recommend overlapping titles, or start conversations. Defaulting to public optimizes for
that discovery loop on first use, when most users haven't yet explored privacy settings.

**Tradeoff acknowledged:**
The cost is privacy-by-default. A user who saves a sensitive or personal pick (e.g., a
guilty-pleasure title) may not realize it's visible until they've already added it. The
alternative — `public=False` by default — is safer for privacy but adds friction for the
common case in a social platform: users would need to opt in to sharing before anyone can
see their list. If CineLog later adds per-entry or per-list visibility controls in the UI,
the default matters less; until then, `public=True` matches the product's stated community
focus, with the understanding that explicit visibility toggles would be a worthwhile follow-up.

---

## Comment 5 — Sort order
> I'd prefer watchlists to default to "date added" order rather than alphabetical. Most
> users want to see what they added recently. I'm open to discussion if you see it
> differently — but let's make a decision and document it.

**My position:**
Adopt date-added order (newest first), matching the maintainer's preference and the existing
collection behavior.

**Reasoning:**
`get_collection()` already sorts by `date_added` descending (newest first). Watchlists serve
a similar "personal queue" purpose — users add films over time and most often care about
what they saved recently, not alphabetical order. Sorting by date added keeps the two list
types consistent and reduces cognitive load when switching between collection and watchlist
views.

**Engagement with reviewer's point:**
The maintainer is right that "most recently added" is the natural question for a watchlist.
Alphabetical order helps when searching for a specific title in a long list, but CineLog
doesn't currently expose search within a watchlist — and even with search, default display
order is about browsing, not lookup. I considered keeping alphabetical as a stable, predictable
order, but it prioritizes catalog structure over user intent. Date-added better reflects why
someone opens their watchlist: "What did I just save?" I updated `get_watchlist()` to use
`.order_by(WatchlistEntry.date_added.desc())` instead of `.order_by(Film.title.asc())`.

---

## Comment 6 — Rebase
> A refactor merged to `main` that changed film IDs from integers to UUIDs. Your
> watchlist code still references integer IDs. Please rebase on `main` and update
> accordingly.

**What conflicted:**
After `git fetch origin && git rebase origin/main`, Git rebased cleanly but the resulting
`models.py` from `main` did not include the `WatchlistEntry` class — the initial watchlist
commit's model changes were effectively lost when combined with main's UUID-refactored models.
Additionally, `WatchlistEntry.film_id` in the original branch was `db.Integer`; `main` migrated
`Film.id` to `db.String(36)` UUID. Docstrings in `watchlist_service.py` and
`routes/watchlist/watchlist.py` still described `film_id` as an integer.

**How I resolved it:**
Re-added `WatchlistEntry` to `models.py` with `film_id = db.Column(db.String(36),
db.ForeignKey("film.id"), nullable=False)` and a `UniqueConstraint` on `(user_id, film_id)`.
Updated docstrings to reference UUID film IDs. The test's fake UUID
(`00000000-0000-0000-0000-000000000000`) already matched the post-refactor schema.

**How I verified no conflict remains:**
Ran `git log --oneline --merges origin/main..HEAD` — no merge commits. Ran `pytest tests/ -v`
— all 5 tests passed. Confirmed `Film.id` and `WatchlistEntry.film_id` are both `String(36)`
in `models.py`. Started the app with `python app.py` and verified the watchlist blueprint
registers without import errors.

---

## PR Description

### Overview

This PR adds a **watchlist** feature to CineLog. Users can save films they plan to watch
(separate from their collection of already-watched films). The implementation includes:

- `WatchlistEntry` model with `user_id`, `film_id`, `date_added`, and `public` fields
- `POST /watchlist/<user_id>/add` — add a film to a user's watchlist
- `GET /watchlist/<user_id>` — retrieve a user's watchlist
- Service layer: `add_to_watchlist()`, `get_watchlist()` with deduplication and error handling

### Design Decisions

1. **Default visibility (`public=True`):** Watchlists default to public because CineLog is a
   community platform where sharing planned viewing supports discovery. Privacy-conscious users
   would benefit from a future per-entry toggle; until then, the default matches the product's
   social focus.

2. **Sort order (date added, newest first):** Watchlists sort by `date_added` descending,
   consistent with `get_collection()` and the maintainer's preference. Users typically want to
   see recently saved films first rather than an alphabetical catalog.

### Manual Testing

1. Start the app: `python app.py` (runs on `http://localhost:5000` by default).
2. Create or identify a user UUID and film UUID (via `/films/` or direct DB seed).
3. **Add to watchlist:**
   ```bash
   curl -X POST http://localhost:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_uuid>"}'
   ```
   Expect HTTP 201 with the new entry JSON.
4. **Duplicate add:** Repeat the same request. Expect HTTP 409 with an error message.
5. **Nonexistent film:** Send a fake UUID as `film_id`. Expect HTTP 404.
6. **Get watchlist:**
   ```bash
   curl http://localhost:5000/watchlist/<user_id>
   ```
   Expect a JSON array sorted with the most recently added film first.
7. **Run tests:** `pytest tests/ -v` — all tests should pass.

---

## Commit History

```
1016458 fix: sort watchlist by date added to match collection behavior
40347ca fix: update WatchlistEntry film_id to UUID after main branch refactor
fa7486c test: add test for nonexistent film_id in add_to_watchlist
6af4402 fix: add deduplication check to prevent duplicate watchlist entries
e1c7995 fix: rename save_to_watchlist to add_to_watchlist per naming convention
b8993ba fix: update film retrieval method to use db.session.get in collection and watchlist services
d738c07 feat: add watchlist model and endpoint
```

*(Screenshot of `git log --oneline origin/main..HEAD` — replace with actual screenshot for submission.)*
