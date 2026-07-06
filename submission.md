# Mixtape — Bug Hunt Submission

## Codebase Map

### Data model (`models.py`)

Six entities, all with UUID string primary keys (`generate_uuid()`):

- **User** — has `listening_streak` (int) and `last_listened_at` (nullable datetime) columns used directly by streak logic (no separate streak table). `friends` is a **symmetric self-referential many-to-many** via the `friendships` table — friendship rows are inserted in both directions (see `seed_data.py add_friendship()`), so `user.friends` works without extra "is this mutual?" logic.
- **Song** — `shared_by` (FK to User) records who originally shared it. `tags` is a many-to-many via `song_tags` (`lazy="subquery"`).
- **ListeningEvent** — one row per (user, song, timestamp). This is the raw material for both the streak service and the feed service.
- **Rating** — one row per (user, song), enforced by a `UniqueConstraint`. Re-rating updates the existing row rather than inserting a new one.
- **Playlist** — songs live in `playlist_entries`, a many-to-many table that (unlike `song_tags`) carries extra columns: `position` (explicit ordering, not insertion order), `added_by`, and `added_at`. This is what makes ordered playlist retrieval and "who added this song" notifications possible.
- **Notification** — flat table: `user_id` (recipient), `notification_type`, `body` (already-rendered text), `read`. Notifications are pre-formatted strings created at write time, not templated at read time.

### Routes → Services

Every route file is a thin adapter: parse the request, call exactly one service function, translate `ValueError` → an HTTP error code (400 for bad input, 404 for missing resource), otherwise `jsonify(...)`. No business logic or DB queries live in `routes/`.

| Route file | Endpoints | Delegates to |
|---|---|---|
| `routes/songs.py` | search, get song, rate, listen | `search_service`, `notification_service.rate_song`, `streak_service.record_listening_event` |
| `routes/playlists.py` | create, get, list songs, add song | `playlist_service`, `notification_service.add_to_playlist` |
| `routes/users.py` | get user, streak, notifications, mark-read | `streak_service.get_streak`, `notification_service` |
| `routes/feed.py` | listening-now, activity | `feed_service` |

### Services (`services/`)

- **`streak_service.py`** — `record_listening_event()` writes a `ListeningEvent` then calls `update_listening_streak()`, which compares `now.date()` to `user.last_listened_at.date()`: same day → no-op, +1 day → increment, gap → reset to 1.
- **`feed_service.py`** — `get_friends_listening_now()` pulls each friend's most recent `ListeningEvent` within a `RECENT_THRESHOLD` window and dedupes to one entry per friend. `get_activity_feed()` is the same shape but unfiltered by recency (just the last N events).
- **`search_service.py`** — `search_songs()` does a case-insensitive `ilike` match on title/artist, `search_service.get_song()` is a plain lookup.
- **`notification_service.py`** — the one place notifications get created. `add_to_playlist()` adds the song to the playlist *and* notifies the original sharer (unless they added it themselves). `rate_song()` creates/updates a `Rating` row. `get_notifications()` / `mark_as_read()` are plain reads/writes.
- **`playlist_service.py`** — `create_playlist`, `get_playlist`, `get_user_playlists` are straightforward CRUD-ish reads. `get_playlist_songs()` joins through `playlist_entries` ordered by `position`.

### Data flow — user rates a song

```
POST /songs/<song_id>/rate
  routes/songs.py: rate()
    → notification_service.rate_song(user_id, song_id, score)
        - validates score 1–5
        - upserts a Rating row (unique on user_id+song_id)
        - commits
        - returns the Rating
  ← 201 {rating}
```

Contrast this with the sibling flow, **add song to playlist**:

```
POST /playlists/<playlist_id>/songs
  routes/playlists.py: add_song()
    → notification_service.add_to_playlist(playlist_id, song_id, added_by)
        - appends song to playlist.songs, commits
        - if adder != song.shared_by: create_notification(shared_by, "song_added_to_playlist", ...)
  ← 201 {message}
```

`add_to_playlist()` calls `create_notification()` for the song's original sharer. `rate_song()` never calls `create_notification()` at all — there's no `"song_rated"` notification type anywhere in the file. This is issue #4: rating a song is silently un-notified while playlist-adding is notified, even though both are "someone interacted with my shared song" events and structurally should follow the same pattern.

### Patterns noticed

- **Routes are dumb, services own everything.** No query or commit ever appears outside `services/`.
- **Errors are `ValueError` → HTTP status**, uniformly, in every route.
- **`notification_service.py` is a hub, not just notification CRUD** — it also contains the playlist-add and rate mutation logic, because notifications are a side effect of those actions rather than a first-class action of their own. Worth remembering when tracing "who else touches Ratings/playlist membership."
- **Two of the three test files pin down exact expected values** (`assert u.listening_streak == 2`, `assert len(songs) == 5`) with comments naming the current buggy result (`# Should be 1, bug causes it to be 3`) — these are effectively pre-written regression tests for 3 of the 5 issues, not just example usage.
- **`seed_data.py` comments double as spec.** E.g. `# Older events (1–14 days ago) — should NOT appear in "listening now" after fix` tells you what the *correct* recency window has to reject, which is more precise than the one-line issue title.

---

## Issue Investigation Notes

I ran the existing test suite (`pytest tests/`) and wrote small manual repros for the two services without test coverage before deciding what to fix. Findings:

| # | Issue | Status | Root cause |
|---|---|---|---|
| 1 | Streak keeps resetting | **Confirmed** — `test_streak_increments_on_sunday` fails (`1 == 2`) | `streak_service.py` `update_listening_streak()`: `elif days_since_last == 1 and today.weekday() != 6` — the `and today.weekday() != 6` clause has no business being there; it forces a reset instead of an increment specifically when the listen happens to fall on a Sunday. |
| 5 | Last song in playlist never shows | **Confirmed** — both playlist tests fail (`4 == 5`, missing `"Track 5"`) | `playlist_service.py` `get_playlist_songs()` returns `songs[:-1]` instead of `songs` — an off-by-one slice that drops the last (correctly-ordered) song every time. |
| 4 | Notified on playlist-add but not on rating | **Confirmed by code inspection** (no test file exists for `notification_service.py`) | `rate_song()` in `notification_service.py` never calls `create_notification()`, unlike the structurally identical `add_to_playlist()`. Needs a `"song_rated"` notification to the song's original sharer, mirroring the `song.shared_by != user_id` self-notify guard already used in `add_to_playlist()`. |
| 2 | Friends Listening Now shows people from yesterday | **Confirmed by manual repro**, not by unit test (no `test_feed.py`) | `feed_service.py`: `RECENT_THRESHOLD = timedelta(hours=24)`. A rolling 24-hour window routinely spans two calendar days, so anything from "last night" still counts as "now." The `seed_data.py` comment explicitly expects events as recent as 2 hours old to be excluded after the fix, which only makes sense if the intended threshold is much shorter than 24h (something on the order of an hour, matching the "past 30 minutes" recent-event fixtures). |
| 3 | Same song shows up twice in search | **Not reproduced** — all 5 `test_search.py` cases pass as-is, including the one commented `# Should be 1, bug causes it to be 3` | `search_service.py` `search_songs()` does an `outerjoin` against `song_tags` with no `.distinct()`/grouping, which *does* produce 3 raw SQL rows for a 3-tag song (verified directly against the compiled SQL). But `db.session.query(Song)` (SQLAlchemy's legacy `Query` API) auto-deduplicates full-entity results by identity, so the ORM-level result is already deduped to 1 before `to_dict()` runs. The join is dead weight / a latent footgun (breaks if this ever gets ported to `session.execute(select(...))` style, which does *not* auto-dedupe), but it isn't the currently-observable bug. **Needs more investigation** — either the real duplicate-causing code path is elsewhere, or this issue is about removing fragile-but-currently-harmless code. Lowest confidence of the five.

### Rough plan

Tackling **#1 (streak), #5 (playlist), #4 (notification)** first — all three have a single, precisely located, high-confidence root cause (two are pinned by failing tests, the third by direct comparison against the working `add_to_playlist` pattern). **#2 (feed threshold)** is next in line and also well-understood. **#3 (search)** needs another pass to find the actual duplication trigger before attempting a fix, since the obvious suspect doesn't reproduce.

---

## Bug Reproductions (before any fix code)

For each of the three chosen bugs, reproduced the exact reported behavior against the real seeded app/DB (not just re-reading the code) before touching any source.

### Issue #1 — streak resets instead of incrementing on Sunday

**How I reproduced it:** The real system clock is currently a Monday, so hitting the live `/songs/<id>/listen` endpoint today can't land on the buggy branch — the bug only triggers when the *second* consecutive listening day is a Sunday. Reproduced deterministically by calling `update_listening_streak()` directly (the same technique `test_streak_increments_on_sunday` uses) against the real seeded `darius` user record, with `last_listened_at`/`listening_streak` reset to a clean start:

```
saturday = 2026-07-04 20:00 UTC  (weekday()==5)
sunday   = 2026-07-05 20:00 UTC  (weekday()==6)

update_listening_streak(darius, saturday)  -> streak = 1
update_listening_streak(darius, sunday)    -> streak = 1   (expected 2)
```

Two calendar-consecutive listens produced no increment solely because the second one fell on a Sunday — confirms the `today.weekday() != 6` clause in `update_listening_streak()` is the trigger. Session rolled back afterward; no persisted changes.

### Issue #5 — last song in a playlist never shows up

**How I reproduced it:** Used the already-seeded "Friday Energy" playlist (`e4378d2b-7000-4ff3-aa94-ea1b0bee0280`, created by darius, 7 songs added by `seed_data.py`) against the live running app:

```
GET /playlists/e4378d2b-7000-4ff3-aa94-ea1b0bee0280/songs
-> {"count": 6, "songs": [...]}   # missing "Harlem Renaissance", position 7
```

Cross-checked directly against the `playlist_entries` table: 7 rows exist for that playlist at positions 1–7. The API dropped exactly the position-7 row — matches `songs[:-1]` in `get_playlist_songs()`.

### Issue #4 — notified on playlist-add but not on rating

**How I reproduced it:** Against the live running app, checked nova's notifications, had a friend (simone) rate one of nova's shared songs, then checked again:

```
GET  /users/211503ef.../notifications          -> count: 1 (existing "song_added_to_playlist" notif)
POST /songs/8db90d75.../rate {"user_id": simone, "score": 5}  -> 201, rating created
GET  /users/211503ef.../notifications          -> count: 1 (unchanged — no new notification)
```

The rating itself was saved successfully (verified in the 201 response), but no notification was generated for nova, the song's original sharer — confirms `rate_song()` in `notification_service.py` never calls `create_notification()`.

---

## Root Cause Analysis

### Issue #1 — My listening streak keeps resetting

**How I reproduced it:** See "Bug Reproductions" above — drove `update_listening_streak()` directly with a Saturday timestamp followed by a Sunday timestamp against the real seeded `darius` record. Streak stayed at 1 after the Sunday listen instead of advancing to 2, even though only one calendar day separated the two listens (the same gap that correctly increments the streak on every other day of the week).

**How I found the root cause:** Started at the route (`routes/songs.py: listen()`) and followed the one call it makes, `streak_service.record_listening_event()`, which itself delegates all the actual logic to `update_listening_streak()`. That's the entire call chain — no other file touches `listening_streak` or `last_listened_at`. Read `update_listening_streak()`'s docstring first ("If the user listened yesterday: streak increments by 1" — no exceptions listed), then read the code and found `elif days_since_last == 1 and today.weekday() != 6`. The `and today.weekday() != 6` clause doesn't correspond to anything in the docstring's stated rules, and a `grep -rn "weekday"` across the whole codebase turned up only this one line plus the test file that names the bug — confirming this wasn't a fragment of some other, legitimate weekly-reset feature living elsewhere. That's the moment I was confident: an undocumented, unreferenced condition on the one line that controls the increment.

**The root cause:** `days_since_last == 1` alone means "the user listened exactly one calendar day ago," which per the docstring should always increment the streak. The code additionally required `today.weekday() != 6` (Python's `datetime.weekday()` returns 6 for Sunday) — meaning the increment branch is skipped, and the function falls through to the `else` clause that resets the streak to 1, on any day the *current* listen happens to be a Sunday, regardless of how consecutive the listening actually was.

**Fix and side-effect check:** Deleted the `and today.weekday() != 6` clause, leaving `elif days_since_last == 1:` as the sole condition for incrementing (`services/streak_service.py`). Ran the full `tests/test_streaks.py` suite afterward: all 5 pass, including the now-fixed `test_streak_increments_on_sunday` **and** the other boundary cases in the same function — same-day no-op (`test_streak_does_not_double_count_same_day`), a genuine skipped-day reset (`test_streak_resets_after_skipped_day`), and the baseline new-user case — confirming the fix only removed the erroneous Sunday special-case without loosening the real boundaries (0-day no-op, >1-day reset). Also ran the full project test suite (`pytest tests/`) to check for any cross-file regression; only the still-unfixed playlist tests failed, nothing new broke.

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it:** See "Bug Reproductions" above — `GET /playlists/e4378d2b-.../songs` on the seeded "Friday Energy" playlist (7 songs, per `seed_data.py`'s `all_songs[3:10]` slice) returned only 6, and a direct query against the `playlist_entries` table confirmed all 7 rows exist at positions 1–7. The response was always missing exactly the position-7 song.

**How I found the root cause:** Followed the route → service chain: `routes/playlists.py: get_songs()` calls `playlist_service.get_playlist_songs()` directly, and that's the only function involved — no other layer touches the result before it's jsonified. Read `get_playlist_songs()` top to bottom: the SQL query itself (join on `playlist_entries`, filter by `playlist_id`, order by `position` ascending) is correct and returns all matching rows. The very next line is `return [song.to_dict() for song in songs[:-1]]`. The docstring right above the function even states "This function returns all songs in the playlist" — a direct contradiction with a `[:-1]` slice sitting one line below the query. That contradiction between the stated contract and the actual return statement was the confirming moment, not just "this looks suspicious."

**The root cause:** `songs` already holds the correctly-ordered, complete result set from the query. The return statement discards the last element of that list unconditionally via `songs[:-1]`, regardless of how many songs are in the playlist. For any non-empty playlist this drops exactly one song — the one in the last position — and for a single-song playlist it silently returns an empty list instead of one song.

**Fix and side-effect check:** Changed `songs[:-1]` to `songs` so the full ordered list is returned (`services/playlist_service.py`). Verified against three sizes to check both boundaries: an empty playlist (`test_empty_playlist_returns_empty_list`, still passes — 0 songs was never affected by this bug), a hand-built single-song playlist (previously would have returned `[]`, now correctly returns the one song — the worst-case symptom of the bug), and the original 7-song "Friday Energy" playlist re-queried live (`Harlem Renaissance` now present). Ran the full test suite (`pytest tests/`, 13/13 passing) to confirm nothing else — search, streaks, or the other playlist helpers (`create_playlist`, `get_playlist`, `get_user_playlists`) — regressed, since they don't touch this slice at all.

### Issue #4 — I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:** See "Bug Reproductions" above — checked nova's notification count, had simone (a friend) rate nova's shared song "Midnight Drive" via `POST /songs/<id>/rate`, then checked again: the rating itself succeeded (201, correct rating payload) but nova's notification count stayed at 1, with no new entry.

**How I found the root cause:** `routes/songs.py: rate()` calls exactly one function, `notification_service.rate_song()`, so that's the only place to look. Read it top to bottom — it validates the score, upserts the `Rating` row, commits, and returns. No call to `create_notification()` anywhere in the function. To confirm this was a *missing* step rather than some other misconfiguration, I compared it against `add_to_playlist()` in the same file, which handles a structurally identical case (a friend interacts with your shared song) and does call `create_notification()`, guarded by `song.shared_by != added_by_user_id`. Seeing that working pattern sitting 30 lines above the broken one — same file, same "notify the sharer unless it's themselves" shape, only present in one of the two functions — was the confirming moment: this isn't a logic error inside an existing notification call, it's an entire step that was never written for ratings.

**The root cause:** `rate_song()` persists the `Rating` and returns without ever constructing a `Notification` for the song's original sharer. There is no `"song_rated"` notification type anywhere in `notification_service.py` — the code path that would create one simply doesn't exist, unlike `add_to_playlist()`, which has the equivalent step for the `"song_added_to_playlist"` type.

**Fix and side-effect check:** Added a `create_notification()` call after the commit in `rate_song()`, guarded by `song.shared_by != user_id` (mirroring `add_to_playlist()`'s self-notify guard) and using the already-fetched `rater` object for the username. Verified against the live seeded app: a friend rating the song now produces a `"song_rated"` notification with the correct sharer as recipient; the same song's owner rating their own song produces no new notification (guard works); the rating endpoint's response body and status code are unchanged. Ran the full test suite (`pytest tests/`, 13/13 passing) — no test file exists for `notification_service.py`, so this was verified entirely by direct reproduction rather than an existing regression test, but the other services (streaks, search, playlists) were unaffected since this function isn't called from any of them.
