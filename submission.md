# Mixtape

Mixtape is a Flask + SQLAlchemy REST API for a social music app: users share songs, rate them, build collaborative playlists, track listening streaks, and see what friends are playing. It's a JSON API without templates or a frontend. Everything is exercised through HTTP endpoints that return JSON.

## Codebase Map

### The layers

1. **`app.py`** — the composition root. A Flask application factory (`create_app`) that configures the DB, initializes the shared `SQLAlchemy` instance (`db`), registers the four blueprints under URL prefixes, and calls `db.create_all()`. The `db` object lives here and is imported everywhere else — this is the seam that ties models and services to the app.

2. **`routes/`** — the HTTP boundary. Four blueprints, one per resource. Routes do exactly two things: parse/validate the request (pull fields off `request.get_json()` or query args, return `400` if required fields are missing) and format the response (`jsonify`, set the status code). They contain **no business logic**. Every route delegates to a service function and translates a raised `ValueError` into a `404` or `400` JSON error.

3. **`services/`** — the business logic. Five modules, each owning one feature domain. This is where all the real work happens: querying, streak math, deduplication, notification creation. The `README` explicitly notes the bugs live here.

`models.py` sits underneath all three, defining the schema.

---

### The files

**`models.py`** — Defines 6 SQLAlchemy models plus 3 association tables. All primary keys are string UUIDs (`generate_uuid`), and timestamps default to timezone-aware UTC.

- `User` — username, email, and streak state (`listening_streak`, `last_listened_at`). Has a self-referential many-to-many `friends` relationship (via the `friendships` join table) that is **explicitly symmetric**: friendship is stored as two rows (A→B and B→A), so it must be inserted in both directions (see `seed_data.py`'s `add_friendship`).
- `Song` — title/artist/album/genre, plus `shared_by` (FK to the sharing user) and an optional `share_note`. `to_dict()` flattens its tags to a list of name strings.
- `Tag` — just an id and unique name; many-to-many with `Song` via `song_tags`.
- `ListeningEvent` — one row per listen (user_id, song_id, listened_at). This is the append-only event log that both streaks and the feed are computed from.
- `Rating` — a user's 1–5 score for a song. A `UniqueConstraint(user_id, song_id)` enforces one rating per user per song — the rating is a first-class model, not a column on `Song`.
- `Playlist` — name, `created_by`, `is_collaborative` flag. Its `songs` relationship goes through the `playlist_entries` association table.
- `Notification` — user_id (recipient), a `notification_type` string, a `body` message, and a `read` flag.

**The 3 association tables** are worth calling out because two of them carry extra columns:
- `friendships` — plain (user_id, friend_id) join, symmetric as described above.
- `song_tags` — plain (song_id, tag_id) join.
- `playlist_entries` — **not a plain join table.** It adds `position` (an explicit integer ordering — songs in a playlist have a defined order, not just insertion order), `added_by` (who added the song), and `added_at`. This is the join-table-with-payload pattern.

**`routes/songs.py`** — `GET /songs/search?q=`, `GET /songs/<id>`, `POST /songs/<id>/rate`, `POST /songs/<id>/listen`. Note the cross-cutting delegation: rating goes to `notification_service.rate_song` (because rating can produce a notification), listening goes to `streak_service.record_listening_event`, and search/detail go to `search_service`.

**`routes/playlists.py`** — `POST /playlists/`, `GET /playlists/<id>`, `GET /playlists/<id>/songs`, `POST /playlists/<id>/songs`. Adding a song delegates to `notification_service.add_to_playlist` (again, because the action notifies someone).

**`routes/users.py`** — `GET /users/<id>`, `GET /users/<id>/streak`, `GET /users/<id>/notifications?unread_only=`, `POST /users/notifications/<id>/read`. This is the one route file that touches the DB directly (`db.session.get(User, ...)` in `get_user`) rather than going through a service — the only exception to the delegation rule.

**`routes/feed.py`** — `GET /feed/<id>/listening-now` and `GET /feed/<id>/activity`. Thin pass-throughs to `feed_service`.

**`services/streak_service.py`** — `record_listening_event` (writes a `ListeningEvent` and updates the streak in one commit), `update_listening_streak` (the day-boundary math: same day → no change, one day gap → increment, larger gap → reset to 1), and `get_streak`.

**`services/feed_service.py`** — `get_friends_listening_now` (friends' listens within a 24h `RECENT_THRESHOLD`, deduplicated to one most-recent song per friend) and `get_activity_feed` (the most recent N events from friends, *not* time-filtered). Both resolve friend IDs from the `user.friends` relationship and bail early with `[]` if the user has no friends.

**`services/search_service.py`** — `search_songs` (case-insensitive `ILIKE` match on title OR artist, joined against tags) and `get_song`.

**`services/notification_service.py`** — the hub for anything that generates a notification. `create_notification` is the shared writer. `add_to_playlist` appends a song to a playlist and notifies the song's original sharer. `rate_song` upserts a `Rating` (updates the score if one exists, else inserts). Plus `get_notifications` and `mark_as_read`. This module owns two responsibilities that could seem unrelated (playlist-add and rating) — they're grouped here because both are *actions on someone else's song that should notify the sharer*.

**`services/playlist_service.py`** — `create_playlist`, `get_playlist_songs` (joins through `playlist_entries` and orders by `position`), `get_playlist` (metadata only), `get_user_playlists`.

**`seed_data.py`** — Drops and recreates all tables, then populates 5 users with friendships, 25 songs (deliberately with 0, 1, and 3+ tags), 3 playlists, listening events spanning recent and old, and a sample notification. Standalone script run via `python seed_data.py`.

**`tests/`** — `test_streaks.py`, `test_search.py`, `test_playlists.py`. Pytest suites targeting the service layer.

---

### Data flow: sharing/rating a song → notification

The clearest end-to-end flow is **rating a song**, which crosses the route → service → model layers and produces a side effect (a notification):

1. **Request** — `POST /songs/<song_id>/rate` with JSON `{"user_id": ..., "score": ...}`.
2. **Route** (`routes/songs.py::rate`) — pulls `user_id` and `score`, returns `400` if either is missing, casts score to `int`, and calls `rate_song(user_id, song_id, score)`. It does no logic beyond parsing.
3. **Service** (`notification_service.py::rate_song`) — validates the score is 1–5, loads the `Song` and rating `User` (raising `ValueError` → the route turns it into a `404`/`400`), then **upserts**: if a `Rating` already exists for this (user, song) pair it updates the score in place; otherwise it creates a new `Rating`. Commits.
4. **Model** — the `Rating` row is written; the `UniqueConstraint(user_id, song_id)` is what makes the upsert necessary (you can't just blindly insert).
5. **Response** — `rating.to_dict()` serialized as JSON with `201`.

The parallel flow for **adding a song to a playlist** (`POST /playlists/<id>/songs` → `notification_service.add_to_playlist`) shows the notification side of this domain fully: it appends the song to the playlist, then — *if the adder isn't the original sharer* — calls `create_notification(user_id=song.shared_by, type="song_added_to_playlist", body=...)`. So the notification recipient is always the **song's original sharer**, and there's a guard so you never get notified about your own action.

To later read those notifications: `GET /users/<id>/notifications` → `notification_service.get_notifications` → ordered-by-recency list of `Notification.to_dict()`.

---

### Patterns worth noting

- **Strict route/service split.** Routes parse and format; services decide. The one deviation is `routes/users.py::get_user`, which reads a `User` straight from the session. Everything else obeys the rule.
- **Errors flow as exceptions.** Services raise `ValueError` for "not found" / "bad input"; routes catch and map to HTTP status codes. There's no custom exception hierarchy — just `ValueError` carrying a message.
- **The DB session is the shared bus.** `db` is defined in `app.py` and imported by every model and service; there's no repository abstraction — services call `db.session` directly.
- **Events are append-only; state is derived.** `ListeningEvent` is a log. Streaks are maintained incrementally on the `User` row, while the feed is computed on-the-fly by querying events. Two different strategies for consuming the same log.
- **Association tables carry data.** `playlist_entries` (position/added_by/added_at) and the symmetric two-row `friendships` model mean relationships aren't just links — the join rows hold meaning, and the code has to respect that (order by position, insert friendship both ways).
- **`to_dict()` everywhere.** Serialization lives on the models, not in a separate schema layer, so every service returns already-serializable dicts and routes just `jsonify` them.
- **Notifications are organized by trigger, not by data type.** Rather than a "ratings service" and a "playlist service" each firing their own notifications, all notification-producing actions on a shared song live together in `notification_service.py`.


<!-- For each of the 3+ bugs you fix, write an entry in your submission doc with all five of these fields:

Issue number and title

How you reproduced it — What steps did you take to confirm the bug exists before touching any code? What inputs, sequence of actions, or data condition triggered the behavior?

How you found the root cause — Which files did you look at? What was your navigation path? What moment made you confident you'd found the right place — not just a suspicious area, but the specific cause?

The root cause — In plain English, explain exactly what was wrong. Not "there was a bug in the streak logic" — explain the specific condition, comparison, or missing step that caused the problem.

Your fix and side-effect check — What did you change and why does that change fix the root cause? What related functionality did you check afterward to confirm you didn't break anything? -->

## Bug 1 — *"My listening streak keeps resetting"*

### How I reproduced it
Called `update_listening_streak` on a Saturday (`2024-06-15`) and then again on the following day, Sunday (`2024-06-16`). Despite the two listens being on consecutive calendar days — which should increment the streak from 1 to 2 — the streak reset to 1. The `test_streak_increments_on_sunday` test in `tests/test_streaks.py` reproduces this exactly: it asserts the streak should be `2` after listening Saturday then Sunday, and that assertion failed before the fix. This matches the user's report ("my listening streak keeps resetting"): any user with an active streak loses it every Sunday.

### How I found the root cause
I opened `services/streak_service.py` and read `update_listening_streak`, then compared its docstring (the stated streak rules) against the actual implementation. The rules list four cases: no prior history → 1, already listened today → no change, listened yesterday → increment, gap larger than a day → reset. Reading the branch at line 73, the increment condition was `days_since_last == 1 and today.weekday() != 6`. The `today.weekday() != 6` clause has no corresponding rule in the docstring — nothing about streaks mentions Sunday. That mismatch between the documented rules and the extra Sunday guard was the moment I was confident this was the specific cause rather than just a suspicious area.

### The root cause
`datetime.weekday()` returns `6` for Sunday (Monday=0 … Sunday=6). The increment branch required both that exactly one day had passed **and** that today was not Sunday (`today.weekday() != 6`). So whenever the current listen fell on a Sunday, the `elif` condition evaluated to `False` even though the user had listened the day before (Saturday). Execution then fell through to the `else` branch, which sets `user.listening_streak = 1` — actively *resetting* the streak instead of incrementing it. The result: every Sunday, a consecutive-day listener's streak was wiped back to 1. The extra `and today.weekday() != 6` condition was spurious and contradicted the documented streak rules.

### Fix and side-effect check
I removed the `and today.weekday() != 6` clause so the branch reads `elif days_since_last == 1:` — a listen exactly one day after the last one now increments the streak on every day of the week, matching the documented rules. 

To confirm I didn't break anything, I ran the full streak suite (`tests/test_streaks.py`): all 5 tests pass, including the previously-failing `test_streak_increments_on_sunday`, and the other cases that guard the surrounding behavior still hold — new user starts at 1 (`test_streak_starts_at_1_for_new_user`), consecutive weekday listens increment (`test_streak_increments_on_consecutive_day`), same-day repeat listens don't double-count (`test_streak_does_not_double_count_same_day`), and a skipped day still resets to 1 (`test_streak_resets_after_skipped_day`). The `days_since_last == 0` and `else`/reset branches were untouched, so removing the Sunday guard only affects the one-day-gap case it was wrongly interfering with.

## Bug 2 — *"Friends Listening Now shows people from yesterday"*

### How I reproduced it
I reseeded the database (`python seed_data.py`) and inspected the seed data setup in `seed_data.py`: it plants two distinct buckets of listening events — "recent" events from within the past 30 minutes (`now - timedelta(minutes=10 + i*5)`) that *should* appear in "listening now," and "older" events from 1–14 days ago that should *not*. Calling `get_friends_listening_now` for a user with friends returned entries whose `listened_at` fell on the previous day, matching the report: people who listened yesterday were surfacing in a feed that's supposed to show who's listening *now*.

### How I found the root cause
I opened `services/feed_service.py` and read `get_friends_listening_now` top to bottom. The query itself is correct — it resolves friend IDs from `user.friends`, filters `ListeningEvent.listened_at >= cutoff`, orders by most-recent, and deduplicates to one song per friend. The comparison direction and the dedup logic are sound, so the bug wasn't mechanical. That pushed me to the one value the whole filter hinges on: `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`, where `RECENT_THRESHOLD = timedelta(hours=24)` at line 13. Cross-referencing that against `seed_data.py`'s own comments — "within the past 30 minutes → should appear," "1–14 days ago → should NOT appear after fix" — made it clear the threshold's *magnitude*, not the code around it, was the specific cause.

### The root cause
`RECENT_THRESHOLD` was set to 24 hours. The filter `listened_at >= now - 24h` therefore admits any listen from the entire previous day, not just listens happening right now. The code does exactly what it says — the defect is that the window's semantics don't match the feature: "Friends Listening Now" implies people actively listening in the moment, but a 24-hour window treats "yesterday" as "now." So a friend who played a song 23 hours ago still qualified, producing the "shows people from yesterday" symptom.

### Fix and side-effect check
I changed `RECENT_THRESHOLD` from `timedelta(hours=24)` to `timedelta(minutes=30)`. This tightens the window to a genuine "now" horizon: it comfortably clears the seed data's recent bucket (listens 10–20 minutes ago still appear) while excluding the 1–14-day-old bucket entirely, so yesterday's listeners drop off.

To confirm I didn't break anything, I reran the full suite (`python -m pytest`): the streak and search suites still pass, and `get_activity_feed` — the other function in this module — was untouched and is intentionally *not* time-filtered, so shrinking `RECENT_THRESHOLD` has no effect on it. I also re-verified against the reseeded data that the recent events still surface while the older ones no longer do.