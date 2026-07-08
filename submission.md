# Project 5: Mixtape Bug Hunt — Submission

## AI Usage

I used an AI assistant (Claude) as a navigation and reasoning partner throughout
this project. The honest short version: AI was most useful for *orienting fast*
and *explaining code I pointed it at*, and least reliable when it reasoned about
behavior without running anything. Every diagnosis in this doc was confirmed by
actually executing the code, not by taking the AI's word for it.

**Codebase navigation (Milestone 1).** I had the AI read the service, route, and
model files and summarize each file's responsibility and the route→service→model
call chain. This is where it was genuinely strong — turning a dozen files into
the "route-thin, service-fat" mental model and the two data-flow traces in the
codebase map. I verified the map against the actual files (e.g. confirming
`playlist_entries` really does carry a `position` column and that friendships are
stored as two directed rows in `seed_data.py`).

**Where the AI's first read was wrong or incomplete — and how I caught it.** This
was the most valuable part of the collaboration:

- **Issue #3 (search duplicates) does not actually reproduce here.** The AI's
  initial read (and the test docstrings) said the `outerjoin(song_tags)` would
  fan out one row per tag and duplicate multi-tag songs. That's the textbook
  diagnosis — but when I *ran* `search_songs("Crown Heights")` against the seed
  data it returned exactly one row, and the search tests passed. Running the code
  contradicted the plausible-sounding explanation. Digging in, the reason is that
  **SQLAlchemy 2.0's ORM de-duplicates full-entity query results by primary
  key**, so the extra joined rows collapse back to one `Song` object. I asked the
  AI to confirm that ORM behavior *after* I'd observed it, not before — and used
  that to consciously choose the three bugs that reproduce cleanly (#1, #4, #5)
  instead of one that's masked in this version.

- **Issue #2 (feed) is similarly masked by the seed data.** The 24h
  `RECENT_THRESHOLD` is too wide for a "listening now" feed, but the feed's
  own "one row per friend" de-duplication hides it for the user I first queried
  (their most recent event is minutes old, so the older ones drop out). Observing
  this steered me away from it as a "clean" fix.

- **A reproduction script crashed for an unrelated reason.** My first combined
  repro tried to exercise `add_to_playlist`, which appends via the relationship
  and can't populate the `NOT NULL` `position` column — an integrity error that
  had nothing to do with the bug I was testing. I isolated Issue #4 down to just
  `rate_song` rather than trusting the bundled script.

**Debugging and tracing (Milestone 3).** For each bug I asked the AI targeted,
post-discovery questions — e.g. "what does `datetime.weekday()` return for
Sunday?" and "what's the structural difference between `add_to_playlist` and
`rate_song`?" — *after* I had already located the suspicious code by tracing the
call chain top-down. The workflow that worked: **I find the code → AI explains a
specific mechanic → I verify by running it with controlled inputs.** I confirmed
every root cause by executing the function directly in an in-memory SQLite app
(isolating the exact triggering variable — weekday for #1, rater≠sharer for #4,
list length for #5) and by running the pytest suite, rather than accepting an
explanation at face value.

**Documentation.** The AI drafted the RCA prose; I kept it honest by only letting
it write claims that matched observed output (the notification body string, the
exact test names and assertion messages, the boundary-check results).

**Things I had to correct the AI on directly:** which virtualenv to use (it
initially created its own and pointed at the wrong parent env), and the fact that
`Mixtape/` is a *cloned* repo with its own `origin` fork — so commits and pushes
had to target that repo's `bugfix/mixtape` branch, not the enclosing directory.

Net: AI accelerated navigation and explanation and drafted clean documentation,
but the diagnoses only became trustworthy once I reproduced each bug by running
the code. The two "obvious" bugs it would have had me fix on inspection (#2, #3)
turned out not to reproduce in this environment — a good reminder of why
reproduce-before-fix is the discipline that mattered most.

---

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

**Symptom:** When a friend adds your shared song to a playlist you receive a
notification, but when a friend rates your shared song you receive nothing —
even though both are "a friend interacted with your song" events.

**How I reproduced it:** State needed: a song whose `shared_by` is user A, and a
*different* user B who rates it. I created sharer A and friend B, shared a song
as A, then called `rate_song(B, song, 5)` and read A's notifications:

- `get_notifications(sharer)` before rating = `0`
- `rate_song` succeeds and stores the rating (score 5)
- `get_notifications(sharer)` after rating = **`0`** (expected 1). ❌

For contrast, the seed data ships a working `song_added_to_playlist`
notification, so the playlist path notifies and the rating path does not.

**How I found the root cause:** This bug is a *missing* action, so instead of
hunting for wrong logic I compared the two interaction paths side by side. Both
live in [notification_service.py](Mixtape/services/notification_service.py). I
traced the working path first: `POST /playlists/<id>/songs` →
`add_to_playlist`, which ends with:

```python
if song.shared_by != added_by_user_id:
    create_notification(user_id=song.shared_by, notification_type="song_added_to_playlist", body=...)
```

Then I traced the broken path: `POST /songs/<id>/rate` in
[routes/songs.py](Mixtape/routes/songs.py) → `rate_song` in the same service.
Reading `rate_song` end to end, it validates the score, upserts the `Rating`,
commits, and `return`s — there is **no `create_notification` call anywhere in
the function**. The moment of confidence: the two functions are structurally
parallel (both load the song, both know `song.shared_by`, both know the acting
user) yet only one contains the notify step. The defect is the absence of that
step in `rate_song`, not a broken condition.

**The root cause:** `rate_song` persists the rating but never notifies the song's
sharer. The notification feature was implemented for the "added to playlist"
interaction (`add_to_playlist` calls `create_notification`) and simply never
wired up for the "rated" interaction. Nothing was computed incorrectly — a
required side effect was omitted entirely, so the sharer's notification list
stays empty after a rating.

**Your fix and side-effect check:** Added a `create_notification(...)` call at
the end of `rate_song`, after the commit, mirroring the proven `add_to_playlist`
pattern: it targets `song.shared_by`, uses type `song_rated`, and a body naming
the rater, song, and score. I reused the same **self-interaction guard**
(`if song.shared_by != user_id`) so rating your own song does not notify you.

Side-effect checks:

- Friend rates sharer's song → sharer gets exactly **1** notification, type
  `song_rated`, body `"friend rated your song 'My Song' 5/5."` ✅
- Sharer rates their **own** song → notification count unchanged (guard
  suppresses self-notification) ✅
- Invalid score (`9`) → still raises `ValueError("Score must be between 1 and
  5")`; the notify step runs only after successful validation and commit ✅
- Full suite: the streak and search tests still pass; the only remaining
  failures are Bug #5's playlist tests (not yet fixed at this point), so this
  change introduced no regressions. ✅

Note on scope: I placed the notify call so it fires on every successful rate by
a non-owner, including re-rating (the model's unique `(user_id, song_id)`
constraint means a re-rate updates in place). Notifying the sharer when a score
changes is consistent with the "friend interacted with your song" intent, and
keeping it unconditional is the smallest change that fixes the reported symptom.

**Commit:** `fix: notify song sharer when a friend rates their song`

---

### Bug #5 — The last song in a playlist never shows up

**Symptom:** Viewing a playlist's songs always shows one fewer song than the
playlist actually contains — specifically, the song in the last position is
missing. An empty playlist still shows empty (no crash), which is why it went
unnoticed as an off-by-one rather than a total failure.

**How I reproduced it:** State needed: a playlist with N songs at positions
1..N. I built a 5-song playlist (`Track 1`..`Track 5`, positions 1–5) and called
`get_playlist_songs`:

- Returned **4** songs: `['Track 1', 'Track 2', 'Track 3', 'Track 4']` —
  `Track 5` (the last position) is dropped. ❌

Reproduced by the failing tests
`tests/test_playlists.py::test_playlist_returns_all_songs` (`4 != 5`) and
`::test_playlist_returns_songs_in_order` ("Right contains one more item:
'Track 5'"). `test_empty_playlist_returns_empty_list` still passes, confirming
it's an off-by-one on the tail, not a broken query.

**How I found the root cause:** Navigation path was route → service, following
the data flow for "view a playlist's songs." `GET /playlists/<id>/songs` in
[routes/playlists.py](Mixtape/routes/playlists.py) calls `get_playlist_songs` in
[playlist_service.py](Mixtape/services/playlist_service.py). I read that function
top to bottom. The query itself is correct — it joins `playlist_entries`, filters
by `playlist_id`, and orders by `position` ascending — so the `songs` list is
complete and correctly ordered *before* the return. The defect is on the return
line itself:

```python
return [song.to_dict() for song in songs[:-1]]
```

The moment of confidence: the docstring explicitly promises "returns all songs
in the playlist," but the comprehension iterates `songs[:-1]`, which is every
element *except the last*. Because the query already ordered by position, "the
last element" is exactly the highest-position (last) song — matching the reported
symptom precisely. That's the specific cause, not just a suspicious spot.

**The root cause:** `get_playlist_songs` correctly retrieves and orders all
playlist songs, but its return statement slices the list with `songs[:-1]`, which
drops the final element. Since the list is ordered by `position` ascending, the
dropped element is always the song in the last position. On an empty playlist
`[][:-1]` evaluates to `[]`, so the empty case looked correct and masked the
off-by-one.

**Your fix and side-effect check:** Changed `songs[:-1]` to `songs` so the
comprehension iterates the full, already-ordered list. This is the minimal
change — the query, join, filter, and `ORDER BY position` were all correct and
left untouched; only the erroneous slice was removed.

Side-effect check — for this boundary bug I checked both ends of the list length
range:

- `pytest tests/test_playlists.py` → all 3 pass: all-songs (now 5), in-order
  (`Track 1..5`), and empty-list.
- Single-song playlist → returns `['Only Track']` (previously the `[:-1]` would
  have dropped the *only* song, returning `[]`) ✅
- Empty playlist → still returns `[]` with no error ✅
- Ordering preserved — songs still come back in ascending `position` order ✅
- Full suite: **13 passed** (streak, search, and playlist), so no regressions.

**Commit:** `fix: stop dropping the last song from playlist song list`

---

### Milestone 2 checkpoint

All three chosen bugs can be triggered deliberately, and I know the exact inputs
/ data conditions for each: **#1** needs a Saturday→Sunday consecutive listen;
**#4** needs a rater who is not the song's sharer; **#5** needs a playlist with
at least one song. No code has been changed.
