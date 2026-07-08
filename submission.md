# Project 5: Mixtape Bug Hunt — Submission

## Codebase Map

Mixtape is a Flask + SQLAlchemy JSON API (no HTML templates — every endpoint
returns JSON). It follows a strict three-layer architecture:

**`app factory` -> `blueprint routes` -> `service functions` -> `SQLAlchemy models`**

Routes never touch the database directly; they parse the request, call one
service function, and format the response. All business logic lives in
`services/`. This means every bug in the tracker traces back to a service file,
with the route as the entry point.

---

### Main files and their roles

| File                                                                         | Role                                                                                                                                                                                                                                                                  |
| ---------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [app.py](Mixtape/app.py)                                                     | Flask application factory (`create_app`). Creates the single `db = SQLAlchemy()` instance, configures the SQLite URI, registers the four blueprints under their URL prefixes (`/songs`, `/playlists`, `/users`, `/feed`), and calls `db.create_all()`.                |
| [models.py](Mixtape/models.py)                                               | All SQLAlchemy models and association tables. Imports `db` from `app`.                                                                                                                                                                                                |
| [seed_data.py](Mixtape/seed_data.py)                                         | Drops and recreates the DB, then populates 5 users, 13 songs, 3 playlists, tags, listening events, and notifications. Comments in here flag which data is meant to expose each bug (e.g. "these are the ones that expose Issue #3").                                  |
| [routes/songs.py](Mixtape/routes/songs.py)                                   | Song search, detail, rating, and "listen" endpoints.                                                                                                                                                                                                                  |
| [routes/playlists.py](Mixtape/routes/playlists.py)                           | Playlist create, detail, list-songs, and add-song endpoints.                                                                                                                                                                                                          |
| [routes/users.py](Mixtape/routes/users.py)                                   | User profile, streak, and notification (list + mark-read) endpoints.                                                                                                                                                                                                  |
| [routes/feed.py](Mixtape/routes/feed.py)                                     | "Friends listening now" and "activity feed" endpoints.                                                                                                                                                                                                                |
| [services/streak_service.py](Mixtape/services/streak_service.py)             | Records listening events and updates each user's consecutive-day listening streak.                                                                                                                                                                                    |
| [services/feed_service.py](Mixtape/services/feed_service.py)                 | Builds the "friends listening now" feed (recency-filtered, one row per friend) and the general activity feed (latest N events).                                                                                                                                       |
| [services/search_service.py](Mixtape/services/search_service.py)             | Case-insensitive song search over title/artist.                                                                                                                                                                                                                       |
| [services/notification_service.py](Mixtape/services/notification_service.py) | Creates and retrieves notifications; also owns `add_to_playlist` and `rate_song` (the two friend interactions that are _supposed_ to notify a song's sharer).                                                                                                         |
| [services/playlist_service.py](Mixtape/services/playlist_service.py)         | Playlist creation and ordered song retrieval.                                                                                                                                                                                                                         |
| [tests/](Mixtape/tests/)                                                     | `pytest` suites for streaks, search, and playlists. Each test file spins up an in-memory SQLite DB via the `create_app({"TESTING": True, ...})` fixture. The test docstrings openly describe the expected buggy behavior (e.g. "Should be 1, bug causes it to be 3"). |

---

### Data model (models.py)

Five entities plus three association tables:

- **`User`** — carries denormalized streak state directly on the row:
  `listening_streak` (int) and `last_listened_at` (datetime). Self-referential
  many-to-many `friends` relationship through the **`friendships`** table.
  Friendships are stored as **two directed rows** per pair (seed inserts both
  `A→B` and `B→A`), so the relationship behaves symmetrically.
- **`Song`** — has `shared_by` (FK to the User who shared it). This field is the
  key to notifications: interactions notify the _original sharer_.
- **`Tag`** + **`song_tags`** — many-to-many tags on songs.
- **`ListeningEvent`** — one row per listen (`user_id`, `song_id`,
  `listened_at`). Feeds and streaks are both derived from these rows.
- **`Rating`** — `(user_id, song_id, score 1–5)` with a **unique constraint on
  `(user_id, song_id)`**, so a user has at most one rating per song (re-rating
  updates in place).
- **`Playlist`** + **`playlist_entries`** — the join table is _not_ a plain
  many-to-many: it adds a **`position`** integer column. Songs in a playlist
  have an explicit order, not just insertion order. It also records `added_by`
  and `added_at`.
- **`Notification`** — `(user_id, notification_type, body, read)`. There is no
  polymorphic link back to the source object; the human-readable message is
  baked into `body` at creation time.

---

### Data flow — feature trace #1: a friend adds your song to a playlist → you get notified

This is the notification path that _works_, and it's the template the broken
paths are supposed to match.

1. `POST /playlists/<playlist_id>/songs` with JSON `{song_id, added_by}` hits
   `add_song()` in [routes/playlists.py](Mixtape/routes/playlists.py). The route
   validates that both fields are present, then delegates.
2. It calls `add_to_playlist(playlist_id, song_id, added_by)` in
   [services/notification_service.py](Mixtape/services/notification_service.py).
3. That service loads the `Song`, the adding `User`, and the `Playlist` (raising
   `ValueError` if any is missing — the route turns that into a 400).
4. If the song isn't already in the playlist, it appends and commits.
5. **The notification step:** if `song.shared_by != added_by_user_id` (i.e. you
   didn't add your own song), it calls `create_notification(...)` targeting
   `song.shared_by` with type `song_added_to_playlist` and a message naming the
   adder, the song, and the playlist.
6. `create_notification` writes a `Notification` row and commits.
7. Later, the sharer calls `GET /users/<user_id>/notifications` →
   `get_notifications()`, which returns their notifications newest-first.

The **self-interaction guard** (`shared_by != actor`) and the **"notify the
sharer" pattern** are the two ideas to carry into the rating path.

### Data flow — feature trace #2: a user listens to a song → streak updates

1. `POST /songs/<song_id>/listen` with `{user_id}` →
   `record_listening_event()` in
   [services/streak_service.py](Mixtape/services/streak_service.py).
2. The service creates a `ListeningEvent` stamped with `now`
   (`datetime.now(timezone.utc)`), then calls `update_listening_streak(user, now)`.
3. `update_listening_streak` compares `now.date()` to `last_listened_at.date()`:
   same day → no change; exactly one day later → increment; otherwise → reset to
   1. It then stamps `last_listened_at = now`.
4. The streak is read back via `GET /users/<user_id>/streak` → `get_streak()`,
   which just returns the denormalized `user.listening_streak`.

---

### Patterns I noticed

- **Route-thin, service-fat.** Every route does exactly three things: parse
  input, call one service function, format the JSON response. Services own all
  logic and all DB access. When an endpoint misbehaves, the route is almost
  never the cause — trace straight to the service.
- **Services raise `ValueError` for "not found"; routes translate it** to a 404
  or 400. This is the uniform error convention across the app.
- **State is denormalized onto the `User` row** for streaks
  (`listening_streak`, `last_listened_at`) rather than being recomputed from
  `ListeningEvent` rows on each read. Fast reads, but the update logic must be
  exactly right or the stored value drifts.
- **Notifications are fire-and-forget and message-baked.** They're created as a
  side effect inside interaction services and store a pre-rendered string. The
  two interaction services (`add_to_playlist`, `rate_song`) _should_ be
  symmetric in how they notify the sharer.
- **Ordering is explicit, not implicit.** `playlist_entries.position` means song
  order is a stored column and must be honored with an `ORDER BY position` on
  read — never rely on insertion order.
- **The seed data and test docstrings are deliberate breadcrumbs.** The seed
  builds songs with 0 / 1 / 3+ tags specifically to surface the search bug, and
  places listening events at specific ages to surface the feed bug. Reading them
  is part of reproducing each issue.

---

## The Five Open Issues (read before choosing)

Per the brief, I read all five issue descriptions before choosing which three to
fix. Summary of where each one lives and my first-pass read of each:

1. **Streak keeps resetting** → `streak_service.update_listening_streak`. There
   is a suspicious `and today.weekday() != 6` on the increment branch — a
   Sunday-specific condition.
2. **Friends Listening Now shows people from yesterday** → `feed_service`. The
   `RECENT_THRESHOLD` is 24 hours, which is far too wide for a "listening _now_"
   feed.
3. **Same song shows up twice in search** → `search_service.search_songs`. It
   `outerjoin`s `song_tags` even though tags aren't used in the filter, which
   fans out one row per tag.
4. **Notified on playlist-add but not on rating** → `notification_service`.
   `add_to_playlist` calls `create_notification`; `rate_song` never does.
5. **Last song in a playlist never shows** → `playlist_service.get_playlist_songs`.
   The return statement slices `songs[:-1]`, dropping the final element.

**Bugs I plan to fix: #1, #4, and #5** — these three reproduce
cleanly and have unambiguous user impact (confirmed by the failing tests for #1
and #5, and by a manual repro for #4, where rating a friend's song produced zero
notifications for the sharer). I'll document reproduction, root cause, fix, and
verification for each in this file.

---

## Root Cause Analysis

Each entry uses five fields: **Symptom**, **How I reproduced it**, **Root
cause**, **The fix**, and **Verification**. Milestone 2 completes the first three
(reproduction is done; no code has been changed yet). *The fix* and
*Verification* are filled in during the fix milestone.

Reproduction environment: `Week5/.venv` (Flask 3, SQLAlchemy 2.0.51,
pytest 9). All three were triggered without touching any code.

---

### Bug #1 — Listening streak keeps resetting

**Symptom:** A user who listens on consecutive days should see their streak
climb. Instead, some users' streaks reset to 1 for no apparent reason. The issue
reporter noticed it "keeps resetting" — the hidden pattern is that it only
happens across a **Saturday → Sunday** transition.

**How I reproduced it:** The condition that triggers the bug is *the second day
being a Sunday* (`date.weekday() == 6`). I drove `update_listening_streak`
directly with fixed dates, using a control case to isolate the Sunday factor:

- Control — Mon 2024-06-10 → Tue 2024-06-11: streak went `1 → 2` ✅ (correct).
- Bug — Sat 2024-06-15 → Sun 2024-06-16: streak stayed at **1** instead of
  going to 2. ❌

The only difference between the two runs is the weekday of the second listen,
which pins the trigger to Sunday. This is also captured by the existing failing
test `tests/test_streaks.py::test_streak_increments_on_sunday` (`assert 1 == 2`).

**How I found the root cause:** Navigation path was route → service, top-down.
The streak is written on the "listen" action, so I started at
`POST /songs/<id>/listen` in [routes/songs.py](Mixtape/routes/songs.py), which
calls `record_listening_event` in
[streak_service.py](Mixtape/services/streak_service.py). That function only
creates the `ListeningEvent` and delegates all streak math to
`update_listening_streak`, so the defect had to live there. Reading that function
line by line, the branch that decides "increment vs reset" is:

```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```

The moment I was confident: `days_since_last == 1` is exactly the
"listened yesterday" case that should *always* increment, yet it's ANDed with
`today.weekday() != 6`. `datetime.weekday()` returns 6 for Sunday, so on Sundays
this branch's condition is `True and False → False`, and control falls through to
the `else` that resets to 1. That's the specific line and comparison producing
the visible bug — not just a "suspicious area."

**The root cause:** Python's `datetime.weekday()` returns 0 for Monday … 6 for
Sunday. The increment branch required `today.weekday() != 6`, so any consecutive
listen whose *second* day landed on a Sunday failed the increment test and fell
into the reset branch. A user with a running streak who listened Saturday and
again Sunday had their streak wiped to 1 instead of incremented — and because it
only misfires one day out of seven, it looked like a random "keeps resetting."
There is no business rule that treats Sunday differently; the `and
today.weekday() != 6` clause is spurious.

**Your fix and side-effect check:** Removed the `and today.weekday() != 6`
clause so the branch is simply `elif days_since_last == 1:`. This is the smallest
change that addresses the root cause — the day-difference arithmetic
(`days_since_last`) was already correct and untouched.

Side-effect check — I verified both sides of the day boundary and the other
streak branches, since this is a boundary-condition bug:

- `pytest tests/test_streaks.py` → all 5 pass (new-user starts at 1, consecutive
  increment, same-day no double-count, skipped-day reset, Sunday increment).
- Manual boundary runs: Sat→Sun = **2** (into Sunday), Sun→Mon = **2** (out of
  Sunday), Sat→Sun→Mon = **3** (three consecutive), Sat→Mon = **1** (skipped
  Sunday still resets correctly), Sunday twice same day = **1** (no double
  count). The reset and same-day paths still work; only the erroneous
  Sunday-reset behavior changed.

**Commit:** `fix: remove spurious Sunday condition from streak increment logic`

---

### Bug #4 — Notified when a friend adds my song to a playlist, but not when they rate it

- **Symptom:** When a friend adds your shared song to a playlist you receive a
  notification, but when a friend rates your shared song you receive nothing —
  even though both are "a friend interacted with your song" events.

- **How I reproduced it:** State needed: a song whose `shared_by` is user A, and
  a *different* user B who rates it. I created sharer A and friend B, shared a
  song as A, then called `rate_song(B, song, 5)` and read A's notifications:
  - `get_notifications(sharer)` before rating = `0`
  - `rate_song` succeeds and stores the rating (score 5)
  - `get_notifications(sharer)` after rating = **`0`** (expected 1). ❌
  For contrast, the seed data ships a working `song_added_to_playlist`
  notification, and `add_to_playlist` visibly calls `create_notification` — so
  the playlist path notifies and the rating path does not.

- **Root cause (identified during repro, not yet fixed):** In
  [notification_service.py](Mixtape/services/notification_service.py),
  `add_to_playlist` ends with a `create_notification(...)` call guarded by
  `if song.shared_by != added_by_user_id`. `rate_song` saves the `Rating` and
  commits but **never calls `create_notification` at all** — the notify step was
  simply omitted from the rating path.

- **The fix:** _(fix milestone)_

- **Verification:** _(fix milestone)_

---

### Bug #5 — The last song in a playlist never shows up

- **Symptom:** Viewing a playlist's songs always shows one fewer song than the
  playlist actually contains — specifically, the song in the last position is
  missing. An empty playlist still shows empty (no crash), which is why it went
  unnoticed as an off-by-one rather than a total failure.

- **How I reproduced it:** State needed: a playlist with N songs at positions
  1..N. I built a 5-song playlist (`Track 1`..`Track 5`, positions 1–5) and
  called `get_playlist_songs`:
  - Returned **4** songs: `['Track 1', 'Track 2', 'Track 3', 'Track 4']` —
    `Track 5` (the last position) is dropped. ❌
  Reproduced by the failing tests
  `tests/test_playlists.py::test_playlist_returns_all_songs` (`4 != 5`) and
  `::test_playlist_returns_songs_in_order` ("Right contains one more item:
  'Track 5'"). `test_empty_playlist_returns_empty_list` still passes, confirming
  it's an off-by-one on the tail, not a broken query.

- **Root cause (identified during repro, not yet fixed):** In
  [playlist_service.py](Mixtape/services/playlist_service.py),
  `get_playlist_songs` builds the correctly-ordered `songs` list but returns
  `[song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice discards the last
  element. (On an empty list, `[][:-1]` is still `[]`, which is why the empty
  case looks fine.)

- **The fix:** _(fix milestone)_

- **Verification:** _(fix milestone)_

---

### Milestone 2 checkpoint

All three chosen bugs can be triggered deliberately, and I know the exact inputs
/ data conditions for each: **#1** needs a Saturday→Sunday consecutive listen;
**#4** needs a rater who is not the song's sharer; **#5** needs a playlist with
at least one song. No code has been changed.
