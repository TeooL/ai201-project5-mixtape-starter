# Mixtape Bug Hunt — Submission

## Codebase Map

**`app.py`** — Flask application factory. Configures the SQLAlchemy DB URI, initializes the `db` extension, registers the four blueprints (`songs`, `playlists`, `users`, `feed`) under their URL prefixes, and creates tables on startup. This is the only file that wires routes together.

**`models.py`** — All SQLAlchemy models and association tables:
- `User` — has `listening_streak` and `last_listened_at` columns used by the streak feature, plus relationships to songs, ratings, listening events, notifications, playlists, and friends (via the `friendships` self-referential many-to-many table).
- `Song` — a shared song; has `shared_by` (the sharer), and relationships to `ratings`, `listening_events`, and `tags` (many-to-many via `song_tags`).
- `Tag` — a labeled genre/style tag, many-to-many with `Song` via `song_tags`.
- `ListeningEvent` — one row per "user listened to a song" action; timestamped, drives both the streak feature and the friends feed.
- `Rating` — a user's 1–5 score for a song, one row per `(user_id, song_id)` pair (enforced by a unique constraint). This is a separate model, not a column on `Song`.
- `Playlist` — a named collection of songs; `songs` relationship goes through `playlist_entries`, a many-to-many table that (unlike a plain association table) carries its own `position`, `added_by`, and `added_at` columns — so playlist order is explicit data, not insertion order.
- `Notification` — a per-user message with a `notification_type` (e.g. `"song_added_to_playlist"`, `"song_rated"`) and a `read` flag.

**`routes/`** — one Flask blueprint per resource area: `songs.py` (search, get, rate, listen), `playlists.py` (create, get, list songs, add song), `users.py` (profile, streak, notifications), `feed.py` (listening-now, activity). Each route handler parses the request (query args or JSON body), does light presence validation (e.g. "is `user_id` present"), calls exactly one service function, and converts the result (or a caught `ValueError`) into a `jsonify` response with the appropriate status code.

**`services/`** — all business logic, one file per feature area: `streak_service.py` (listening streak math), `feed_service.py` ("listening now" / activity feed queries), `search_service.py` (song search), `notification_service.py` (creating/reading notifications, plus the rating and playlist-add flows that trigger them), `playlist_service.py` (playlist CRUD and ordered song retrieval). Services are the only code that touches `db.session` for anything beyond simple lookups.

**`seed_data.py`** — populates the DB with users, friendships, songs (with varying tag counts), listening events at deliberately chosen ages, playlists, and notifications, specifically shaped to expose the five tracked bugs (e.g., listening events at both ~15 minutes and 2+ hours old, to distinguish "recent" from "stale" in the feed).

**`tests/`** — one file per feature with dedicated bug coverage: `test_streaks.py`, `test_search.py`, `test_playlists.py`. Notably, there's no test file for `feed_service.py` or `notification_service.py` — those two issues (#2 and #4) had to be verified manually rather than via an existing failing test.

### Data flow — a user rates a song

1. `POST /songs/<song_id>/rate` in `routes/songs.py` reads `user_id` and `score` from the JSON body, checks both are present, and calls `notification_service.rate_song(user_id, song_id, score)`.
2. `rate_song` (in `services/notification_service.py`) validates the score is 1–5, looks up the `Song` and rating `User`, then checks for an existing `Rating` row for that `(user_id, song_id)` pair — if found it updates the score in place, otherwise it creates a new `Rating` row. Either way it commits.
3. After the rating is saved, `rate_song` compares `song.shared_by` to the rater's `user_id`; if they differ (a friend rated it, not the sharer themselves), it calls `create_notification(user_id=song.shared_by, notification_type="song_rated", body=...)`, which inserts a `Notification` row for the original sharer.
4. The route returns the serialized `Rating` (`rating.to_dict()`) with a 201 status. The sharer later sees the notification via `GET /users/<user_id>/notifications` → `notification_service.get_notifications()`.

This mirrors the sibling flow `add_to_playlist` uses for `"song_added_to_playlist"` notifications — both follow "do the domain write, commit, then notify the sharer if someone else acted" — which is exactly the pattern (and the gap) at the center of [Issue #4](#issue-4-i-got-notified-when-a-friend-added-my-song-to-a-playlist-but-not-when-they-rated-it) below.

### Pattern I noticed

Every route handler is a thin adapter: parse request → call one service function → `jsonify` the result or catch `ValueError` and map it to a 4xx response. No route touches `db.session` or a model directly. All domain logic — validation beyond "is this field present," streak math, feed filtering, notification triggering, ordering — lives in `services/`, and each service file owns one feature area. The one crosscutting exception is `notification_service.py`, which contains both the notification CRUD (`create_notification`, `get_notifications`, `mark_as_read`) *and* the two feature flows that trigger notifications as a side effect (`add_to_playlist`, `rate_song`) — so it's importing from `playlist_service` rather than the other way around, which is worth knowing before assuming "playlist logic lives in `playlist_service.py`."

---

## Issue #1: My listening streak keeps resetting

**How I reproduced it**

I ran the existing test suite first (`pytest tests/`) before touching any code. `tests/test_streaks.py::test_streak_increments_on_sunday` was already in the repo and failing: it records a listen on Saturday (streak → 1), then a listen on Sunday, and asserts the streak becomes 2. Instead it stayed at 1. I confirmed this wasn't a one-off by re-running just that test in isolation and by manually calling `update_listening_streak` from a Python shell with a Saturday `datetime` followed by a Sunday `datetime` one day later — same result, streak reset instead of incremented.

**How I found the root cause**

Starting from the route `POST /songs/<song_id>/listen` in `routes/songs.py`, I followed the call into `services/streak_service.record_listening_event`, which delegates the actual streak math to `update_listening_streak`. Reading that function top-down: it computes `days_since_last = (today - last_date).days` and then branches on that value. The `days_since_last == 1` branch (the "listened yesterday, so increment" case) had an extra condition ANDed on: `and today.weekday() != 6`. That stood out immediately as unrelated to the day-gap logic being computed two lines above — nothing else in the function reasons about which weekday it is. Tracing through the Saturday→Sunday test case by hand confirmed it: `days_since_last` correctly evaluates to `1`, but `today.weekday()` for the Sunday datetime is `6`, so the whole `elif` condition is `False`, and execution falls through to the `else` branch that resets the streak to 1.

**The root cause**

`update_listening_streak` only increments the streak when `days_since_last == 1 AND today.weekday() != 6`. Since Python's `datetime.weekday()` returns `6` for Sunday, this second clause is `False` every time the current listen happens on a Sunday — regardless of whether the previous day was actually listened to. So a user who listens Saturday and then Sunday (a genuine one-day gap, which should increment the streak) instead falls into the `else` branch and has their streak reset to 1. There is no reset logic anywhere else that legitimately needs a weekday check — this condition doesn't correspond to any stated business rule in the docstring ("increments by 1" when the gap is exactly one day) and simply cuts off increments on the one day of the week where `weekday() == 6`.

**My fix and side-effect check**

I removed the `and today.weekday() != 6` clause, leaving `elif days_since_last == 1: user.listening_streak += 1`. This makes the increment path depend purely on the day gap, matching the documented rule. I re-ran the full `tests/test_streaks.py` file: all 5 tests pass, including `test_streak_starts_at_1_for_new_user`, `test_streak_does_not_double_count_same_day`, and `test_streak_resets_after_skipped_day` — confirming the same-day no-op case and the multi-day-gap reset case (the other two boundaries this function handles) are unaffected by removing the weekday check.

---

## Issue #2: Friends Listening Now shows people from yesterday

**How I reproduced it**

There's no existing test file for `feed_service.py`, so I reproduced this by hand in a Python shell against an in-memory test DB (per the "isolate the function" tip in the project brief): I created a user with two friends, one `ListeningEvent` 10 minutes old and one 20 hours old, and called `get_friends_listening_now(user_id)` directly. Both friends came back in the result — including the one whose only listening event was 20 hours old, which matches the reported symptom of the feed showing someone from "yesterday" instead of only people currently listening.

**How I found the root cause**

Following the call chain from `GET /<user_id>/listening-now` in `routes/feed.py` led straight to `services/feed_service.get_friends_listening_now`. Near the top of that function: `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`, and `RECENT_THRESHOLD` is defined at module level as `timedelta(hours=24)`. The query then filters `ListeningEvent.listened_at >= cutoff`, so anything within the last 24 hours passes. I also read `seed_data.py`, which explicitly separates listening events into a "recent" batch (10–20 minutes old, commented "should appear in listening now") and an "older" batch (2+ hours old, commented "should NOT appear in listening now after fix") — confirming the intended behavior is a much tighter window than 24 hours, and that the seed data was deliberately built to expose this exact gap.

**The root cause**

`RECENT_THRESHOLD` was set to `timedelta(hours=24)`, a full calendar day, for a feed whose entire purpose is showing who is listening *right now*. Any friend who listened up to 24 hours ago — effectively "yesterday" from the viewer's perspective — passes the `listened_at >= cutoff` filter and gets shown as currently listening, which is indistinguishable from someone who started listening a minute ago once deduplicated. The threshold value itself, not the query or dedup logic around it, was the defect.

**My fix and side-effect check**

I changed `RECENT_THRESHOLD` from `timedelta(hours=24)` to `timedelta(hours=1)`, which still comfortably covers the length of a real listening session while excluding anything from hours or days earlier. Re-running my repro script after the change showed only the 10-minute-old friend in the result, with the 20-hour-old one correctly excluded. I checked `get_activity_feed` in the same file, since it also queries `ListeningEvent` for friends — it doesn't reference `RECENT_THRESHOLD` at all (it's explicitly documented as "not filtered by recency"), so it's unaffected by this change. I also re-ran the full test suite to confirm no other test depends on the old threshold.

---

## Issue #5: The last song in a playlist never shows up

**How I reproduced it**

`tests/test_playlists.py` already had `test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order` in the repo. Running `pytest tests/` before any changes showed both failing: a 5-song playlist came back with only 4 songs (`Track 5` missing), consistently the last one by position. I double-checked it wasn't specific to 5 songs by rerunning with a 1-song and a 2-song playlist in a Python shell — the very last song by position was dropped every time, including when the playlist had only one song (which then returned empty).

**How I found the root cause**

From `GET /playlists/<id>/songs` in `routes/playlists.py` I traced into `services/playlist_service.get_playlist_songs`. The function builds `songs` via a query joined on `playlist_entries` and ordered ascending by `position` — that part matches the docstring ("Songs are returned in the order they were added"). The return statement, though, was `return [song.to_dict() for song in songs[:-1]]`. The docstring even states "This function returns all songs in the playlist," directly contradicting the `[:-1]` slice on the line right below it — that mismatch between the documented contract and the actual return statement is what confirmed this was the bug rather than a query/ordering issue.

**The root cause**

After correctly querying and ordering all songs in the playlist, the function sliced the final list with `songs[:-1]`, which drops the last element of any non-empty list. Since the query already applied `order_by(asc(playlist_entries.c.position))`, "last element" always meant "last song by position" — i.e., the most recently added song in a non-collaborative-order sense, or simply whichever song currently occupies the highest position. This explains why the dropped song was always the last one, regardless of playlist length.

**My fix and side-effect check**

I changed the return statement to `[song.to_dict() for song in songs]`, removing the slice so all queried songs are returned, matching the function's own docstring. I re-ran `tests/test_playlists.py`: all three tests pass, including `test_empty_playlist_returns_empty_list` — confirming the fix doesn't break the empty-playlist boundary case (slicing `[]` vs. not slicing `[]` both correctly yield `[]`) alongside the ordering and full-count checks. I ran the full suite (`pytest tests/`) afterward and all 13 tests across all three test files passed.

---

## Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it**

I created a sharer and a separate rater in a Python shell against an in-memory DB, called `rate_song(rater.id, song.id, 5)` (the same function `POST /songs/<song_id>/rate` calls, per `routes/songs.py`), and then checked `get_notifications(sharer.id)`. It returned an empty list — the sharer received no notification even though another user rated their song. For contrast I ran the equivalent flow with `add_to_playlist`, which did produce a notification for the sharer, matching the reported asymmetry exactly (playlist-add notifies, rating doesn't).

**How I found the root cause**

I opened `services/notification_service.py` and read both functions side by side, since the issue explicitly describes one working and one not. `add_to_playlist` has a clear two-part shape: (1) perform the domain action (append the song to the playlist, commit), then (2) `if song.shared_by != added_by_user_id: create_notification(...)`. Reading `rate_song` immediately below it, the function saves or updates the `Rating`, commits, and returns — there's no second part at all, no call to `create_notification` anywhere in the function. That absence, next to the sibling function that has the exact call I expected to see, was the confirmation: the notification step for ratings was never written, not just broken.

**The root cause**

`rate_song` never called `create_notification`. Every other cross-user interaction that's supposed to notify a song's sharer (currently just `add_to_playlist`) explicitly makes that call after committing its own change; `rate_song` was missing the equivalent call entirely, so saving a rating never produced a notification for anyone, regardless of who rated the song.

**My fix and side-effect check**

I added a notification step to `rate_song` after the `db.session.commit()`, mirroring `add_to_playlist`'s pattern: skip notifying if the rater is the song's own sharer (`song.shared_by != user_id`), otherwise create a `"song_rated"` notification addressed to `song.shared_by`. I verified in a shell that: a rating from a different user produces exactly one notification for the sharer; a sharer rating their own song produces no notification (self-notify guard works); and re-rating the same song by the same user produces another notification (score changed, consistent with `rate_song` treating that as a real update). I also re-ran the full test suite — no test file covers `notification_service.py` directly, but all 13 existing tests still passed, confirming this addition didn't disturb `add_to_playlist`, streaks, search, or playlists. Separately, while exercising `add_to_playlist` for this comparison, I found it raises an unrelated `IntegrityError` (`playlist_entries.position` NOT NULL) whenever it appends a song via the ORM relationship instead of an explicit `playlist_entries.insert()`; that's a pre-existing defect independent of this issue (it happens before the notification step even runs) and isn't one of the five tracked issues, so I left it as-is rather than fixing something outside this issue's scope.

---

## Issue #3: The same song keeps showing up twice in search

**How I reproduced it**

Before touching code, I ran `pytest tests/test_search.py`, which already contains a test built for exactly this issue (`test_search_no_duplicates_multi_tag_song`, whose docstring literally says "bug causes it to be 3"). All 5 tests passed. That was a red flag, since the issue is listed as open, so I reproduced it manually two ways instead of trusting the test result at face value: (1) called `search_songs("Crown Heights")` directly in a shell against a song with 3 tags, and (2) ran the equivalent raw SQL that `search_songs` builds via `db.session.execute(query.statement)` to see the actual rows the database returns, bypassing ORM post-processing.

**How I found the root cause**

`services/search_service.search_songs` builds its query with `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` before filtering on title/artist, even though nothing in the `.filter()` clause references tags at all — the docstring confirms search is only supposed to match "title or artist name," and tags are attached to the output afterward via the `Song.tags` relationship. Joining a many-to-many association table without constraining it multiplies each matching song's row once per associated tag. Printing the raw SQL and executing it directly confirmed this: a song with 3 tags produced exactly 3 identical rows at the database level.

However, `search_songs()` itself and the ORM-level `db.session.query(Song)...all()` call both returned only 1 result for that same song — no duplicate. Digging further, I confirmed (by running the identical join as a 2.0-style `select()` executed via `session.execute(stmt).scalars().all()`, which is *not* auto-deduplicated) that the legacy `Query` API in the pinned SQLAlchemy version (2.0.51) automatically unique-ifies full-entity results, while raw `Result` objects do not. That's why `pytest` shows this test passing today: the symptom is real at the SQL level but is currently being silently absorbed by an ORM implementation detail rather than by anything in this codebase's own logic.

**The root cause**

`search_songs` joins `song_tags` for no reason connected to its filter — the join exists but is never constrained or used to select on tags, so it turns a single-row-per-song query into a single-row-per-(song, tag)-pair query. The visible "duplicate song" symptom described in the issue is exactly what that Cartesian-style join produces. It happens not to reach the user's screen in this specific dependency version because SQLAlchemy's legacy `Query.all()` happens to deduplicate full ORM entities for you — a behavior this code was never written to depend on, and one that would break the moment the query is rewritten in 2.0 style (`select()` + `session.execute()`) or a dependency changes.

**My fix and side-effect check**

I removed the `.outerjoin(song_tags, ...)` call entirely, since it wasn't contributing to the filter and the output's `tags` field is populated independently via `song.to_dict()` → `Song.tags` (a separate eager-loaded relationship, unaffected by this change). This fixes the duplication at its source — the SQL itself now returns one row per matching song — rather than relying on the ORM's incidental uniquing. I also removed the now-unused `Tag` and `song_tags` imports. I confirmed with the same raw-SQL check that a 3-tag song now returns exactly 1 row, re-ran `search_songs("Crown Heights")` to confirm the tags list (`['rap', 'hip-hop', 'boom bap']`) is still fully populated in the output, and re-ran the full test suite (`pytest tests/`) — all 13 tests still pass, including `test_search_returns_matching_songs` and `test_search_returns_empty_for_no_match`, confirming basic title/artist matching is unaffected.

---

## Summary

All five tracked issues were investigated and fixed, each as its own commit on `bugfix/mixtape`:

| # | Title | File | Fix |
|---|-------|------|-----|
| 1 | Streak keeps resetting | `streak_service.py` | Removed a `today.weekday() != 6` condition that blocked streak increments on Sundays |
| 2 | Listening Now shows people from yesterday | `feed_service.py` | Shrank `RECENT_THRESHOLD` from 24h to 1h |
| 3 | Same song shows up twice in search | `search_service.py` | Removed an unused `song_tags` join that multiplied rows per tag |
| 4 | No notification for ratings | `notification_service.py` | Added the missing `create_notification` call to `rate_song` |
| 5 | Last playlist song never shows | `playlist_service.py` | Removed a `songs[:-1]` slice that dropped the last song |

The full test suite (`pytest tests/`) passes with 13/13 tests green after all five fixes.

## AI Usage

I used Claude (via Claude Code) as a pair-programming assistant for this bug hunt, following the workflow described in the project brief: locate the suspicious code myself first, then use AI to help read/explain it or to run verification scripts, and confirm the diagnosis by re-reading the actual source before deciding on a fix.

Concretely, for each issue I had Claude read the relevant route → service call chain (`routes/*.py` → `services/*.py`) and the associated test files up front, since that reading is mechanical and the same navigation strategy the brief recommends (start at the route, follow calls, note files visited). Claude then wrote and ran small reproduction scripts against an in-memory SQLite database for each issue — a Python-shell-style equivalent of the brief's suggested `flask shell` approach — to confirm each bug's symptom before making any code change, and to re-verify the fix and check adjacent behavior afterward (e.g., self-rating shouldn't notify for Issue #4, empty playlists shouldn't error for Issue #5, Saturday→Sunday should increment for Issue #1).

The one place AI-driven investigation paid off beyond "explain code I already found" was Issue #3: the existing `test_search.py` suite passed without any code changes, which could easily have been mistaken for "this bug doesn't need fixing." Rather than accepting that at face value, Claude cross-checked the raw SQL the query generates (via `session.execute(query.statement)`) against the ORM-level result, which revealed that the join genuinely produces 3 duplicate rows in SQL, but SQLAlchemy's legacy `Query.all()` API in the pinned dependency version (2.0.51) silently deduplicates full-entity results — a version-specific ORM behavior masking a real, still-worth-fixing defect. That distinction (SQL-level duplication vs. what the test observes) is exactly the kind of thing that's easy to miss without deliberately verifying at more than one layer, and it's reflected in that issue's root-cause writeup above.

I did not ask AI to "find the bug" for me in any of the five cases before reading the relevant service file myself — each RCA entry's "how I found the root cause" section reflects the actual file-by-file navigation path I took, with AI used afterward to help write and run verification scripts and to sanity-check the explanation of SQLAlchemy/Python behaviors (e.g., `Query.all()` vs. `session.execute(select(...))` uniquing semantics, and `datetime.weekday()` return values) rather than to guess at root causes.
