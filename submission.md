# submission.md — Mixtape Codebase Map

## AI Usage

I used an AI coding assistant (OpenCode) throughout this project. Here's how:

**Codebase orientation.** I asked the AI to read every file in the project and produce a summary of what each module is responsible for. It generated the file-by-file breakdown in section 1 below, which saved me from having to manually reconcile routes against services. I verified the summaries by reading the source files myself afterward — the AI's descriptions were accurate, but I caught one error: it initially described Bug #3 as reproducible ("multi-tag songs return duplicates") when in reality SQLAlchemy 2.x deduplicates by primary key identity and the test actually passes. The AI didn't catch this itself; I had to run `pytest` and see the test pass before realizing the bug doesn't manifest in this version.

**Data flow tracing.** I asked the AI to trace the full call chain for "a user rates a song" (section 2). It walked from the Flask route through the service functions, identifying every validation check, database query, and the missing notification step (Bug #4). This was helpful for understanding the architecture, but the AI didn't flag that `notification_service.py` has a dead import of `get_playlist_songs` on line 45 — I discovered that myself while doing the side-effect check for Bug #5.

**Bug reproduction.** I asked the AI to write short Python scripts that exercise each service function directly (the same pattern the tests use) to reproduce bugs without needing to start the Flask server. It correctly reproduced Bugs #1, #2, #4, and #5, and correctly confirmed that Bug #3 does not reproduce. For Bug #1 (Sunday streak), the AI's explanation of `datetime.weekday()` returning 6 for Sunday and therefore making `weekday() != 6` evaluate to `False` was precise enough to give me confidence in the fix before I touched any code.

**Understanding `models.py`.** I asked the AI to explain association tables and `db.relationship()` calls to someone with basic SQL knowledge. It explained the difference between a FK column (a real DB constraint in one table) and a `relationship()` (an ORM convenience for Python navigation), broke down what `backref` does, and explained the self-referential `User.friends` relationship including what `primaryjoin`, `secondaryjoin`, and `lazy="dynamic"` mean. This saved me time vs. reading the SQLAlchemy docs from scratch.

**App startup confusion.** When the README said "never use `python app.py`" due to a SQLAlchemy double-import bug, I asked the AI to explain why. It traced the problem: `python app.py` creates a `__main__` module, the debug reloader re-imports it as `app`, and you get two `db = SQLAlchemy()` instances. `FLASK_APP=app:create_app flask run` avoids this because the module is always imported with the same canonical name. This clarified a confusing instruction.

**Where the AI was wrong or incomplete:**
- It initially claimed Bug #3 was still reproducible and suggested adding `.distinct()` as the fix. I had to run the test myself to see it passes.
- For Bug #2, the AI reproduced the bug but the fix (`timedelta(minutes=30)`) is a judgment call — there's no "correct" threshold specified in the requirements. I accepted 30 minutes as reasonable but recognize it's arbitrary.
- The AI's initial submission.md described Bug #3's test as "currently failing" when it wasn't. I corrected this.

**What I verified myself:**
- Every `pytest` run to confirm which tests fail and, after fixes, that all pass
- Side-effect checks: traced every call site of the changed functions to confirm nothing else depended on the broken behavior
- The `[:-1]` edge case for empty playlists (`[][:-1]` returns `[]`, same as `[]`) — I wanted to confirm the fix didn't break empty-list handling
- Git workflow: I made my own decisions about staging, committing, and whether to split commits

## 1. File-by-File Breakdown

### `app.py` — Flask application factory and DB setup
- Creates `db = SQLAlchemy()` at module level (single shared instance)
- `create_app(config=None)` is the factory: configures SQLite URI, initializes `db`, registers four blueprints (`songs`, `playlists`, `users`, `feed`), and calls `db.create_all()`
- Every test creates its own app via `create_app()` with `TESTING=True` and an in-memory database; every blueprint registration is a lazy import inside the factory to avoid circular imports
- `if __name__ == "__main__"` block runs `app.run(debug=True)` as a convenience shortcut — but `FLASK_APP=app:create_app flask run --debug` is the correct way for hot-reload dev work (avoids a Flask-SQLAlchemy double-import bug)

### `models.py` — SQLAlchemy models for every database entity
- `generate_uuid()` helper returns `str(uuid.uuid4())` — all PKs are string UUIDs
- **3 association tables** (bridge tables for many-to-many relationships):
  - `friendships` — user ↔ user (self-referential, symmetric)
  - `song_tags` — song ↔ tag
  - `playlist_entries` — playlist ↔ song, with extra columns: `position` (ordering), `added_by` (who added it), `added_at` (when)
- **6 model classes**: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`
- Key relationships on `User`:
  - `shared_songs` / `shared_by_user` (backref) — a user has many songs they shared
  - `ratings` / `rater` — a user has many ratings they submitted
  - `listening_events` / `listener` — a user has many listening events
  - `notifications` / `recipient` — a user has many notifications
  - `playlists` / `creator` — a user has many playlists they created
  - `friends` — self-referential many-to-many through `friendships`, `lazy="dynamic"` means it returns a query object instead of a pre-fetched list
- Key relationships on `Song`:
  - `shared_by` (FK column) + `shared_by_user` (backref) — points to the User who shared it
  - `tags` — many-to-many through `song_tags`, `lazy="subquery"` (eager-loads tags with the song in a single query)
  - `ratings`, `listening_events` — one-to-many backrefs
- `Rating` has a `UniqueConstraint(user_id, song_id)` — one rating per user per song
- Every model has a `to_dict()` method returning a plain dict with `.isoformat()` on datetime fields

### `routes/songs.py` — Song search, rating, and listening endpoints
- `GET /songs/search?q=...` → `search_service.search_songs()`
- `GET /songs/<song_id>` → `search_service.get_song()`
- `POST /songs/<song_id>/rate` → `notification_service.rate_song()` — expects `{"user_id", "score"}`
- `POST /songs/<song_id>/listen` → `streak_service.record_listening_event()` — expects `{"user_id"}`

### `routes/playlists.py` — Playlist CRUD and song management
- `POST /playlists/` → `playlist_service.create_playlist()`
- `GET /playlists/<playlist_id>` → `playlist_service.get_playlist()` (metadata only)
- `GET /playlists/<playlist_id>/songs` → `playlist_service.get_playlist_songs()` (**Bug #5 lives here**)
- `POST /playlists/<playlist_id>/songs` → `notification_service.add_to_playlist()` — adds a song and notifies the sharer

### `routes/users.py` — User profiles, streaks, and notifications
- `GET /users/<user_id>` — direct DB lookup (no service layer — the only exception)
- `GET /users/<user_id>/streak` → `streak_service.get_streak()`
- `GET /users/<user_id>/notifications?unread_only=true` → `notification_service.get_notifications()`
- `POST /users/notifications/<notification_id>/read` → `notification_service.mark_as_read()`

### `routes/feed.py` — Friends Listening Now and activity feed
- `GET /feed/<user_id>/listening-now` → `feed_service.get_friends_listening_now()` (**Bug #2** — threshold too wide)
- `GET /feed/<user_id>/activity` → `feed_service.get_activity_feed()`

### `services/streak_service.py` — Listening streak logic (**Bug #1**)
- `record_listening_event(user_id, song_id)` — creates a `ListeningEvent`, calls `update_listening_streak()`, commits
- `update_listening_streak(user, now)` — core logic: compares `today.date()` to `last_listened.date()`; same day = no change, yesterday = increment, more than one day gap = reset to 1. **Bug #1 at line 73**: `days_since_last == 1 and today.weekday() != 6` prevents increment on Sundays (weekday 6)
- `get_streak(user_id)` — simple lookup returning `user.listening_streak`

### `services/feed_service.py` — Friends Listening Now and activity feed (**Bug #2**)
- `RECENT_THRESHOLD = timedelta(hours=24)` at module level — this is the bug. "Listening now" should be ~30 minutes, not 24 hours
- `get_friends_listening_now(user_id)` — gets all friends' listening events within `RECENT_THRESHOLD`, deduplicates to one song per friend, returns list of `{"friend": ..., "song": ..., "listened_at": ...}`
- `get_activity_feed(user_id, limit=20)` — returns most recent N listening events from friends regardless of recency

### `services/search_service.py` — Song search logic (**Bug #3**)
- `search_songs(query)` — queries `Song` with `.outerjoin(song_tags)`, filters on `title.ilike(...)` or `artist.ilike(...)`. **Bug #3**: the outer join produces one result row per tag per matching song, so a song with 3 tags appears 3 times in `.all()`. Fix: add `.distinct()` or remove the join (tags are eager-loaded via `lazy="subquery"` anyway)
- `get_song(song_id)` — simple `db.session.get(Song, song_id)` with `ValueError` on not found

### `services/notification_service.py` — Notification creation and retrieval (**Bug #4**)
- `create_notification(user_id, type, body)` — creates a `Notification` record and commits
- `add_to_playlist(playlist_id, song_id, added_by_user_id)` — appends the song to `playlist.songs`, then creates a notification for the song's original sharer (if the adder isn't the sharer). This one works correctly
- `rate_song(user_id, song_id, score)` — validates score 1–5, upserts a `Rating`, commits. **Bug #4**: never calls `create_notification()` for the song's sharer. The playlist-add path has the notification; the rating path doesn't
- `get_notifications(user_id, unread_only)` — queries notifications, optionally filtered to `read=False`, ordered by `created_at DESC`
- `mark_as_read(notification_id)` — sets `read=True` and commits

### `services/playlist_service.py` — Playlist creation and retrieval (**Bug #5**)
- `create_playlist(name, created_by_user_id, is_collaborative)` — creates and commits a `Playlist`
- `get_playlist_songs(playlist_id)` — joins `Song` → `playlist_entries`, filters by `playlist_id`, orders by `position ASC`. **Bug #5 at line 66**: returns `songs[:-1]` (drops the last song). Should be `songs`
- `get_playlist(playlist_id)` — returns playlist metadata without songs
- `get_user_playlists(user_id)` — returns all playlists for a user

### `seed_data.py` — Database seeder
- `seed()` function: drops all tables, recreates, then populates 5 users (with friendships), 10 tags, 13 songs (0/1/3+ tags each), listening events (both recent and old), 3 playlists, 1 seed notification. Designed to expose all five bugs so students can test fixes against real data.
- Friendship insertion is explicitly bidirectional (inserts user_id↔friend_id in both directions)
- Playlist entries are inserted via raw `playlist_entries.insert()` (not ORM navigation) to set `position` explicitly

### `tests/test_streaks.py` — 5 tests
- `test_streak_starts_at_1_for_new_user` — first-ever listen sets streak=1
- `test_streak_increments_on_consecutive_day` — Monday→Tuesday bumps to 2
- `test_streak_does_not_double_count_same_day` — two listens on same day = 1
- `test_streak_resets_after_skipped_day` — Monday→Wednesday resets to 1
- `test_streak_increments_on_sunday` — Saturday→Sunday should increment; was **failing before fix** (Bug #1)

### `tests/test_search.py` — 5 tests (all pass)
- `test_search_returns_matching_songs` — basic search by artist works
- `test_search_no_duplicates_single_tag_song` — one-tag songs appear once
- `test_search_no_duplicates_multi_tag_song` — multi-tag songs appear once (SQLAlchemy 2.x deduplicates by primary key identity; Bug #3 does not manifest in this version)
- `test_search_no_duplicates_no_tag_song` — zero-tag songs appear once
- `test_search_returns_empty_for_no_match` — no matches = empty list

### `tests/test_playlists.py` — 3 tests
- `test_playlist_returns_all_songs` — expects 5 songs; was **failing before fix** with 4 (Bug #5)
- `test_playlist_returns_songs_in_order` — validates positional ordering
- `test_empty_playlist_returns_empty_list` — empty playlist returns `[]`

---

## 2. Data Flow: User Rates a Song

```
POST /songs/<song_id>/rate   {"user_id": "...", "score": 4}
│
▼
routes/songs.py::rate()
  ├── Validates that user_id and score are present in request body
  ├── Calls notification_service.rate_song(user_id, song_id, int(score))
  │     ├── Validates score is 1–5
  │     ├── Looks up Song by song_id; raises ValueError if missing
  │     ├── Looks up User by user_id; raises ValueError if missing
  │     ├── Queries for existing Rating WHERE user_id = ? AND song_id = ?
  │     │     ├── If found: updates existing.score = score
  │     │     └── If not found: creates new Rating(...) and db.session.add(rating)
  │     ├── db.session.commit()
  │     ├── BUG #4: does NOT create a notification for song.shared_by
  │     └── Returns the Rating object
  ├── Returns jsonify(rating.to_dict()), 201
  └── Catches ValueError → returns jsonify({"error": str(e)}), 400
```

**What's missing (Bug #4):** `add_to_playlist()` has this pattern at lines 65–70 — it checks `if song.shared_by != added_by_user_id:` and calls `create_notification(...)`. `rate_song()` should do the same: after `db.session.commit()`, check if the rater is not the sharer, and create a `"song_rated"` notification for the song's original sharer. The fix is a ~4 line addition mirroring the playlist notification pattern.

---

## 3. Architectural Patterns

1. **Strict layer separation (Flask factory → ORM → services → blueprints).** Routes never contain business logic. They parse inputs, call a service function, and return JSON. Services own all database access and logic. The only exception is `GET /users/<user_id>` which does a direct `db.session.get()` in the route — everything else goes through a service.

2. **Services raise `ValueError`; routes catch it.** There is no custom exception hierarchy. Every service function raises `ValueError("descriptive message")` for both not-found (404) and validation (400) cases. Routes use the same `except ValueError as e:` catch-all and decide the status code based on context (404 for missing resources, 400 for bad input).

3. **UUIDs as strings everywhere.** All primary keys are `db.String(36)` populated by `generate_uuid()`. No auto-increment integers. FK columns reference these string UUIDs.

4. **Timezones are handled defensively.** All timestamps are created with `datetime.now(timezone.utc)`. `update_listening_streak()` has a guard clause checking `last_listened.tzinfo is None` and replacing with UTC if a naive datetime somehow snuck in.

5. **Models own their serialization.** Every model has `to_dict()` returning a plain dict with `.isoformat()` on datetime fields. Route handlers call `.to_dict()` before `jsonify()`. No separate serializer layer.

6. **Association tables bridge many-to-many relationships.** Three `db.Table` objects at module level handle join logic. `playlist_entries` goes beyond a pure bridge by carrying its own data (`position`, `added_by`, `added_at`) — it's a "rich join table" pattern.

7. **Tests use in-memory SQLite and call services directly.** Each test file has its own `app` fixture creating a fresh in-memory database. Tests run inside `with app.app_context():` and exercise service functions (not HTTP endpoints). Fixtures create test data, yield it so tests can use it, then `db.drop_all()` cleans up in teardown.

8. **Lazy vs eager loading matters.** `Song.tags` uses `lazy="subquery"` (eager — loads tags with the song in one query). `User.friends` uses `lazy="dynamic"` (returns a query object you can chain). Most other relationships use `lazy=True` (load-on-access). Bug #3 happens because `.outerjoin(song_tags)` adds a JOIN that defeats the eager-load and produces duplicate rows.

9. **Module docstrings on every file.** Each `.py` file opens with a `"""filename — Mixtape\n\nDescription."""` docstring. Service functions use Google-style `Args:`/`Returns:` docstrings. Test functions have short descriptive docstrings.

10. **Two of the five services (`notification_service.py`, `playlist_service.py`) are inter-dependent.** `notification_service.add_to_playlist()` imports `playlist_service.get_playlist_songs()` inside the function body (lazy import) to avoid circular imports. The other three services are self-contained.

---

## 4. Bug Fixes — Root Cause Analysis

### Bug #1 — Listening streak resets on Sunday

**Service file:** `services/streak_service.py`, line 73

**How I reproduced it:**
I ran the existing test `test_streak_increments_on_sunday` in `tests/test_streaks.py`, which calls `update_listening_streak()` first with a Saturday datetime (2024-06-15, `weekday()==5`) and then with a Sunday datetime (2024-06-16, `weekday()==6`). The Saturday listen set the streak to 1. The Sunday listen should have incremented it to 2, but the assertion `assert u.listening_streak == 2` failed — the streak was 1. This means the `elif` branch that handles consecutive-day increment was skipped, and the `else` (reset-to-1) branch ran instead.

```
pytest tests/test_streaks.py::test_streak_increments_on_sunday
# FAILED: assert 1 == 2
```

**How I found the root cause:**
The bug description said "streak resets on Sunday." I opened `services/streak_service.py` and read `update_listening_streak()`. The core logic is a three-branch `if/elif/else` on `days_since_last`. The `elif` handles the "listened yesterday → increment" case. But it had an extra condition: `and today.weekday() != 6`. I checked Python's docs — `datetime.weekday()` returns 0=Monday through 6=Sunday. So `weekday() != 6` is `False` on Sunday. This meant the `elif` was unreachable on Sundays, forcing the `else` (reset-to-1) branch to run instead. The moment I saw `weekday() != 6` paired with the failing test that used a Sunday date, I knew this was the exact line.

**Root cause:**
The condition on line 73 is:
```python
elif days_since_last == 1 and today.weekday() != 6:
```
Python's `datetime.weekday()` returns 0 for Monday through 6 for Sunday. When `today` is a Sunday, `today.weekday()` returns 6, making `today.weekday() != 6` evaluate to `False`. The entire `elif` branch is skipped, so execution falls through to the `else` block which resets the streak to 1. The `!= 6` guard has no valid purpose — there's nothing special about Sunday in a listening streak; it's just another day.

**Fix and side-effect check:**
```diff
-    elif days_since_last == 1 and today.weekday() != 6:
+    elif days_since_last == 1:
```
Removing the `and today.weekday() != 6` clause makes Sunday increment like any other consecutive day. Side-effect check: the `weekday()` call appeared nowhere else in any service or route file — only in this one condition. I verified that non-Sunday consecutive days still increment (`test_streak_increments_on_consecutive_day` passes), skipped days still reset (`test_streak_resets_after_skipped_day` passes), and same-day double-counting is still prevented (`test_streak_does_not_double_count_same_day` passes). No other code depends on `weekday()` or Sunday-specific behavior.

---

### Bug #2 — Friends Listening Now shows people from yesterday

**Service file:** `services/feed_service.py`, line 13

**How I reproduced it:**
I created two users (alice and bob) who are friends. Bob has a `ListeningEvent` from 2 hours ago. I called `get_friends_listening_now(alice.id)`. The function returned Bob's 2-hour-old event. A 2-hour-old listen is clearly not "listening now" — this should return an empty list.

```python
now = datetime.now(timezone.utc)
old_event = ListeningEvent(
    user_id=bob.id, song_id=song.id,
    listened_at=now - timedelta(hours=2)
)
feed = get_friends_listening_now(alice.id)
# feed had 1 entry (bob) — BUG: 2-hour-old event should NOT appear
```

The seed data also exposes this: `seed_data.py` creates listening events "within the past 30 minutes" (meant to be recent) and others starting at 2 hours back. With a 24-hour threshold, ALL of them qualify.

**How I found the root cause:**
The bug description said "Friends Listening Now shows people from yesterday." I opened `services/feed_service.py` and saw `RECENT_THRESHOLD = timedelta(hours=24)` at the module level on line 13. That was immediately suspicious — "listening now" with a 24-hour window isn't "now" at all. I traced its only usage on line 32: `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`. This `cutoff` is used as the `>=` comparison filter for `ListeningEvent.listened_at`. So any friend who listened within 24 hours qualifies — including people who listened yesterday. The `get_activity_feed()` function (the other endpoint in this service) doesn't use `RECENT_THRESHOLD` at all, confirming this was the only place the problem could live.

**Root cause:**
```python
RECENT_THRESHOLD = timedelta(hours=24)
```
The `cutoff` in `get_friends_listening_now()` is `datetime.now() - RECENT_THRESHOLD`, then used as the filter `.filter(ListeningEvent.listened_at >= cutoff)`. With `hours=24`, any friend who listened at any time in the past 24 hours appears in "listening now." The endpoint's name and purpose imply a much shorter window — users who listened within the last ~30 minutes.

**Fix and side-effect check:**
```diff
- RECENT_THRESHOLD = timedelta(hours=24)
+ RECENT_THRESHOLD = timedelta(minutes=30)
```
Side-effect check: `RECENT_THRESHOLD` is only used on one line (line 32) in `get_friends_listening_now()`. The sibling function `get_activity_feed()` has no time cutoff at all — it returns the most recent N events regardless of age. Changing this constant does not affect it. I verified with two events: a 10-minute-old listen (appears, correct) and a 2-hour-old listen (excluded, correct). Old events are properly filtered out, recent events still show up.

---

### Bug #5 — The last song in a playlist never shows up

**Service file:** `services/playlist_service.py`, line 66

**How I reproduced it:**
I ran the existing test `test_playlist_returns_all_songs` in `tests/test_playlists.py`, which seeds a playlist with 5 songs (positions 1–5) and then calls `get_playlist_songs()`. The assertion `assert len(songs) == 5` failed — only 4 songs were returned. `test_playlist_returns_songs_in_order` also failed because the expected list `["Track 1", "Track 2", "Track 3", "Track 4", "Track 5"]` was missing "Track 5".

```
pytest tests/test_playlists.py::test_playlist_returns_all_songs
# FAILED: assert 4 == 5
```

The seed data in `seed_data.py` also exposes this: playlists have 7 songs inserted with positions 1–7, but `get_playlist_songs()` returns only 6.

**How I found the root cause:**
The bug description said "the last song in a playlist never shows up." I opened `services/playlist_service.py` and read `get_playlist_songs()`. The query at lines 58–64 looked correct — `.join(playlist_entries)`, filter by `playlist_id`, order by `position ASC` — the SQL side was fine. But the return statement on line 66 was `[song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice immediately stood out — it explicitly drops the last element of the list. The docstring even contradicted it, saying "This function returns all songs in the playlist." That mismatch between the docstring (all songs) and the code ([:-1]) was the confirmation.

**Root cause:**
```python
return [song.to_dict() for song in songs[:-1]]
```
The `[:-1]` is a Python slice that means "all elements except the last one." If the query returns `[Track1, Track2, Track3, Track4, Track5]`, the slice reduces it to `[Track1, Track2, Track3, Track4]`. The test `test_empty_playlist_returns_empty_list` still passed because `[][:-1]` returns `[]` — an empty list with the last element removed is still empty. So the bug only manifests when there's at least one song.

**Fix and side-effect check:**
```diff
-    return [song.to_dict() for song in songs[:-1]]
+    return [song.to_dict() for song in songs]
```
Side-effect check: `get_playlist_songs()` is called by the `GET /playlists/<id>/songs` route and by the playlist tests. It is *imported* (but never called) in `notification_service.py::add_to_playlist()` — dead code with no impact. I verified the empty playlist edge case still works (`test_empty_playlist_returns_empty_list` passes — `[]` with no slice is still `[]`), and both the all-songs and order tests pass. The fix is purely removing an incorrect slice; nothing else in the codebase depends on the last-song-being-dropped behavior.

---

### Bugs NOT chosen (and why)

| Bug | Why not chosen |
|-----|---------------|
| #3 (search duplicates) | Does **not** reproduce. `db.session.query(Song).outerjoin(song_tags).all()` deduplicates by primary key identity in SQLAlchemy 2.x. The test `test_search_no_duplicates_multi_tag_song` actually passes. The `.outerjoin()` is still unnecessary (tags are eager-loaded via `lazy="subquery"`), but the claimed duplicate-row bug doesn't manifest. |
| #4 (rate notification) | A valid bug — `rate_song()` at `notification_service.py:108` never calls `create_notification()` for the song sharer. I reproduced it: after bob rates alice's song, `db.session.query(Notification).filter_by(user_id=alice.id).all()` returns `[]`. The fix mirrors the pattern in `add_to_playlist()` at lines 65–70 (check `if song.shared_by != rater_id` and call `create_notification()`). I chose #2 instead because it has the same character as the other bugs — a one-line logic fix rather than a missing feature. |
