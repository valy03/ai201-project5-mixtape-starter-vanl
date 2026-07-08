# Mixtape — Project Submission

## 1. Codebase Map

Mixtape is a **Flask JSON API** (no HTML frontend) for a music-sharing social app. Users share songs, rate them, add them to collaborative playlists, follow friends' listening activity, and build listening streaks. Data is stored in SQLite via SQLAlchemy.

### Architecture at a glance

```
HTTP request
   │
   ▼
routes/*.py        ← parse input, call a service, format the JSON response
   │
   ▼
services/*.py      ← all business logic (queries, rules, side-effects)
   │
   ▼
models.py          ← SQLAlchemy table definitions + to_dict() serializers
   │
   ▼
SQLite (mixtape.db)
```

The app is strictly **three-layered**: routes never touch the database directly (with one small exception, noted below), and models never contain business logic. `seed_data.py` sits outside this stack and populates the database for testing.

### Main files and their responsibilities

| File | Responsibility |
|------|----------------|
| **`app.py`** | The application factory. `create_app()` configures the SQLite URI and secret key, initializes SQLAlchemy, registers the four blueprints under URL prefixes (`/songs`, `/playlists`, `/users`, `/feed`), and calls `db.create_all()`. Also runnable directly via `python app.py`. |
| **`models.py`** | Defines all 6 entity tables as SQLAlchemy models — `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification` — plus 3 many-to-many association tables (`friendships`, `song_tags`, `playlist_entries`). Every model except `Tag` has a `to_dict()` method that produces its JSON representation. IDs are random UUID strings, not integers. |
| **`seed_data.py`** | Standalone script (`python seed_data.py`) that drops and recreates all tables, then inserts 5 users with friendships, 25 songs (deliberately with 0, 1, and 3+ tags), 3 playlists, listening events (some recent, some old), streaks, and a sample notification. |
| **`routes/songs.py`** | Song endpoints: search, detail, rate, and record-a-listen. |
| **`routes/playlists.py`** | Playlist endpoints: create, detail, list songs, add song. |
| **`routes/users.py`** | User endpoints: profile, streak, list notifications, mark notification read. |
| **`routes/feed.py`** | Feed endpoints: "friends listening now" and "activity feed." |
| **`services/search_service.py`** | `search_songs()` (title/artist substring match with tags) and `get_song()` (single song by ID). |
| **`services/notification_service.py`** | The notification + interaction hub: `create_notification()`, `add_to_playlist()` (adds a song **and** notifies the sharer), `rate_song()`, `get_notifications()`, `mark_as_read()`. |
| **`services/playlist_service.py`** | `create_playlist()`, `get_playlist()`, `get_playlist_songs()` (ordered by position), `get_user_playlists()`. |
| **`services/streak_service.py`** | `record_listening_event()` (creates the `ListeningEvent` **and** updates the streak), `update_listening_streak()` (consecutive-day logic), `get_streak()`. |
| **`services/feed_service.py`** | `get_friends_listening_now()` (friends' plays within a recency window, deduped per friend) and `get_activity_feed()` (most recent N friend plays, no recency filter). |

### The data model in detail

- **`User`** — `username`, `email`, `listening_streak`, `last_listened_at`. Has a self-referential many-to-many `friends` relationship through the `friendships` table.
- **`Song`** — `title`, `artist`, `album`, `genre`, `shared_by` (FK to the User who shared it), `share_note`. The `shared_by` field is what makes notifications possible — it records original ownership.
- **`Tag`** — just a `name`. Linked to songs via `song_tags`. (The only model with no `to_dict()`.)
- **`ListeningEvent`** — `user_id`, `song_id`, `listened_at`. This is the raw event stream that **both** the feed and streak features are derived from. There is no "feed" table — feeds are computed live by querying these events.
- **`Rating`** — `user_id`, `song_id`, `score` (1–5), with a `UniqueConstraint` on (user, song) so a user can rate a song only once (re-rating updates the existing row).
- **`Playlist`** — `name`, `created_by`, `is_collaborative`. Songs are attached through `playlist_entries`, which is a **rich join table** carrying `position` (explicit ordering), `added_by`, and `added_at` — so playlist order is intentional, not insertion order.
- **`Notification`** — `user_id` (recipient), `notification_type`, `body`, `read` flag.

---

## 2. Data Flow — Adding a song to a playlist triggers a notification

This is the app's clearest example of an interaction producing a notification. When a user adds someone else's song to a playlist, the **original sharer** gets notified.

```
POST /playlists/<playlist_id>/songs
  body: { "song_id": "...", "added_by": "..." }
        │
        ▼
routes/playlists.py :: add_song()                         [playlists.py:43-54]
   • extracts song_id + added_by from JSON
   • returns 400 if either is missing
   • calls ↓ (wraps ValueError as 400)
        │
        ▼
notification_service.py :: add_to_playlist(playlist_id, song_id, added_by)   [notification_service.py:35-70]
   1. db.session.get(Song, song_id)      → verify song exists
   2. db.session.get(User, added_by)     → verify the adder exists
   3. db.session.get(Playlist, ...)      → verify playlist exists
   4. if song not already in playlist:   playlist.songs.append(song); commit()
   5. if song.shared_by != added_by:     ← only notify if adder isn't the sharer
        └─► create_notification(
                user_id       = song.shared_by,       ← recipient is the ORIGINAL sharer
                notification_type = "song_added_to_playlist",
                body          = "<adder> added your song '<title>' to '<playlist>'."
            )
        │
        ▼
notification_service.py :: create_notification(...)       [notification_service.py:13-32]
   • builds a Notification row, adds it, commits
```

The recipient later reads it via a **separate** flow:

```
GET /users/<user_id>/notifications
        │
        ▼
routes/users.py :: notifications()   →   get_notifications(user_id)   [notification_service.py:113-128]
   • queries Notification WHERE user_id = <user>, newest first
```

**Key insight:** the notification recipient is `song.shared_by`, *not* the playlist owner. The `shared_by` field on `Song` (set at seed time) is the linchpin — without it there'd be no one to notify. The self-notification guard at [notification_service.py:65](services/notification_service.py#L65) prevents pinging users about their own actions.

The **rating** flow is similar in shape (`POST /songs/<id>/rate` → `rate_song()`), and the **listen** flow (`POST /songs/<id>/listen` → `record_listening_event()`) feeds both the streak system and the live feeds — but notably, neither rating nor listening currently generates a notification, even though the service docstring implies notifications fire "when friends interact with a user's shared songs."

---

## 3. Patterns I noticed

1. **Thin routes, fat services.** Every route follows the same template: parse the request body/args, validate required fields (returning `400`), call one service function inside a `try/except ValueError`, and `jsonify` the result. All real logic — queries, rules, commits — lives in `services/`. The one deviation is `routes/users.py`, which touches `db.session.get(User, ...)` directly in `get_user()` instead of going through a service.

2. **Errors flow up as `ValueError`.** Services raise `ValueError` for "not found" / validation problems; routes catch it and convert to an HTTP error (usually `404` for reads, `400` for writes). This keeps HTTP concerns out of the service layer.

3. **Everything is derived from events, not stored as state.** There is no feed table and no "currently listening" flag. Feeds and streaks are both *computed* by querying `ListeningEvent`. This is elegant but means feed correctness depends entirely on how those queries filter and window the events.

4. **UUID string primary keys everywhere**, generated by `generate_uuid()`. This means IDs can't be guessed — testing endpoints requires reading real IDs out of the seeded DB.

5. **`to_dict()` is the serialization contract.** Every model owns its JSON shape. Routes never hand-build response dicts from model fields; they call `to_dict()` (or a service that does). `Song.to_dict()` flattens tags to a list of names, sidestepping the missing `Tag.to_dict()`.

6. **Services occasionally cross-call each other**, e.g. `add_to_playlist()` calls `create_notification()`, and `record_listening_event()` calls `update_listening_streak()`. Related side-effects are bundled into a single service call rather than orchestrated at the route layer.

### Observations flagged for the bug-fixing phase

Reading the code surfaced several spots that look like planted bugs (the seed file even references "Issue #3" and "Issue #4" by number):

- **`get_playlist_songs()` returns `songs[:-1]`** — [playlist_service.py:66](services/playlist_service.py#L66) — silently drops the **last** song of every playlist.
- **`search_songs()` outer-joins `song_tags`** without de-duplicating — [search_service.py:25-35](services/search_service.py#L25-L35) — so a song with 3 tags appears **3 times** in results (matches the "Issue #3" seed comment).
- **Feed recency threshold is 24 hours** — [feed_service.py:13](services/feed_service.py#L13) — but the seed comment at [seed_data.py:111](seed_data.py#L111) says "listening now" should mean the past 30 minutes.
- **Streak won't increment on Sundays** — [streak_service.py:73](services/streak_service.py#L73) — the `today.weekday() != 6` condition looks like a bug, not a business rule.
- **Unused import** — `add_to_playlist()` imports `get_playlist_songs` but never uses it — [notification_service.py:45](services/notification_service.py#L45).


## The Issues

### Issue #1 — My listening streak keeps resetting
**Reported by:** kenji

I listen to something on Mixtape every single day — I haven't missed a day in weeks. On Saturday night my streak was at 12. Sunday morning I played a song like always, checked my profile, and my streak said 1. This is the second time it's happened, and both times it was a Sunday. Listening again on Monday bumped it to 2, so it's counting again — it just threw away my whole streak.

**Steps I took:**

1. Listened to a song every calendar day, including Saturday.
2. Listened again Sunday morning and checked my streak (`GET /users/<my_id>/streak`).

**Expected:** streak goes from 12 to 13 — I listened on consecutive days.
**Actual:** streak shows 1, as if I'd skipped a day.

**How I reproduced it:**

The bug reads the current date via `datetime.now()`, so it only fires on a real Sunday — a naive API call on any other day looks fine. To trigger it on demand, I called the streak logic directly and *injected* the date, since `update_listening_streak(user, now)` takes `now` as a parameter.

In the Flask shell (project root, venv active, `python`):

```python
from datetime import datetime, timezone
from app import create_app
from models import User
from services.streak_service import update_listening_streak

app = create_app(); app.app_context().push()

# Data condition: a user mid-streak whose last listen was "yesterday" (Saturday)
saturday = datetime(2026, 7, 4, 20, 0, tzinfo=timezone.utc)   # Saturday
sunday   = datetime(2026, 7, 5, 8, 0, tzinfo=timezone.utc)    # the next day, Sunday

kenji = User(username="kenji", listening_streak=12, last_listened_at=saturday)
update_listening_streak(kenji, sunday)
print(kenji.listening_streak)     # prints 1  ← bug (should be 13)
```

**Trigger conditions (all three required):**
1. `days_since_last == 1` — a genuine consecutive-day listen (last listened the day before).
2. The new listen lands on a **Sunday** (`now.weekday() == 6`).
3. Any pre-existing streak value (here 12) — it gets wiped to 1 regardless.

**Control that isolates the trigger:** repeating the exact same one-day gap but landing on a Friday (`thursday → friday`) instead correctly returns `13`. The *only* changed variable is the day of week, which pins the defect to the `and today.weekday() != 6` clause at `services/streak_service.py:73`.

**How I found the root cause:**

1. Started at the entry point for a listen: `POST /songs/<song_id>/listen` in `routes/songs.py`, which calls `record_listening_event()`.
2. Followed that into `services/streak_service.py`. `record_listening_event()` creates the `ListeningEvent` and delegates all streak math to `update_listening_streak(user, now)` — so the streak logic lives entirely in that one function.
3. Read the function's docstring first (its spec): four rules, all defined purely by the day-*gap* between listens. None mention the day of the week.
4. Read the branch logic underneath and immediately saw the mismatch: the increment `elif` carried an extra `and today.weekday() != 6` clause that had no counterpart anywhere in the spec.
5. **The moment of confidence** wasn't just spotting a suspicious line — it was the control experiment. Holding the day-*gap* constant (1 day) and changing only the day of week flipped the result from `1` (Sunday) to `13` (Friday). Since `weekday()` is the function's *only* reference to day-of-week, that single-variable swing proved line 73 was the specific cause, not merely a suspect.

**Root cause:**

Python's `date.weekday()` returns `6` for Sunday (Monday = 0 … Sunday = 6). The streak-increment branch was written as `elif days_since_last == 1 and today.weekday() != 6:` — i.e. "increment only if this is a consecutive-day listen **and** today is not Sunday." Because this `elif` sits in an if/elif/else chain whose `else` branch is the catch-all reset (`user.listening_streak = 1`), the compound condition being `False` on Sundays didn't just skip the increment — it fell through to the reset. So a user who listened Saturday then Sunday (a genuine consecutive-day streak) had `days_since_last == 1` evaluate `True` but `weekday() != 6` evaluate `False`, making the whole `elif` `False`, dropping them into the `else` and wiping their streak to 1. The day-of-week check has no basis in the streak spec; it was extraneous logic that corrupted an otherwise-correct gap calculation.

**Fix and side-effect check:**

Removed the day-of-week clause so the increment depends only on the day-gap, matching the spec:

```python
# services/streak_service.py:73
- elif days_since_last == 1 and today.weekday() != 6:
+ elif days_since_last == 1:
      user.listening_streak += 1
```

This fixes the root cause because a consecutive-day listen now satisfies the `elif` on every day of the week, so it lands in the `+= 1` branch instead of falling through to the reset. Side-effects checked afterward:
- **Same-day re-listen** (`days_since_last == 0`) — still returns early at the first `if`; untouched.
- **Skipped 2+ days** (`days_since_last > 1`) — the `elif` was already `False` (gap ≠ 1), so it still correctly falls to the `else` reset; the removed clause never affected this path.
- **Consecutive listen on a non-Sunday** — previously worked, still works (Thursday→Friday control still returns `13`).
- Re-ran the reproduction: Case A (Sat→Sun) now prints `13` instead of `1`. No imports orphaned — `datetime`/`timezone` are still used elsewhere in the file.

### Issue #2 — Friends Listening Now shows people from yesterday
**Reported by:** nova

"Friends Listening Now" is supposed to show me what my friends are playing right now — or at least what they've played today. This morning around 9am it showed darius "listening now" to a song he told me he played at 11pm last night, before he went to bed. He hadn't opened the app all morning. Stuff from yesterday evening keeps hanging around in the feed until the same time the next day.

**Steps I took:**

1. Opened my feed in the morning (`GET /feed/<my_id>/listening-now`).
2. Cross-checked with darius: his last listen was the previous night.

**Expected:** only friends who have listened today appear.
**Actual:** friends whose last listen was yesterday evening still show up the next morning.

**How I reproduced it:**

Like the streak bug, this is time-dependent: the feed compares each listen against `now`, and the `/listen` endpoint always stamps a listen as the current moment — so you can't trigger "a listen from last night" through the live API. You have to set up the **data condition**: a friend whose most recent listen was several hours ago and who hasn't listened since.

First I confirmed the default seed *doesn't* show the bug — nova's three friends all have listens from the last ~30 minutes, so they legitimately belong in the feed (dedup keeps each friend's most recent listen). To expose the bug I recreated nova's exact story in the Flask shell: gave darius a single listen 10 hours ago and cleared his fresher seeded events.

```python
from datetime import datetime, timedelta, timezone
from app import create_app, db
from models import User, Song, ListeningEvent
from services.feed_service import get_friends_listening_now

app = create_app(); app.app_context().push()
now = datetime.now(timezone.utc)
nova   = User.query.filter_by(username="nova").first()
darius = User.query.filter_by(username="darius").first()
song   = Song.query.first()

ListeningEvent.query.filter_by(user_id=darius.id).delete()
db.session.add(ListeningEvent(user_id=darius.id, song_id=song.id,
                              listened_at=now - timedelta(hours=10)))
db.session.commit()

shown = {r["friend"]["username"] for r in get_friends_listening_now(nova.id)}
print("darius shown as 'listening NOW'?", "darius" in shown)   # -> True  (bug)
```

**Trigger condition:** a friend whose latest listen is between "a while ago" and 24 hours ago (here 10 h) still appears in "listening now."

**Control that isolates the threshold:** re-running with the listen at **25 hours** ago prints `False` — darius drops out. Holding everything else constant and moving only the listen's age across the 24-hour mark flips the result, which pins the defect to the recency window (`RECENT_THRESHOLD`), not to the query or dedup logic.

_(Re-seed with `python seed_data.py` afterward, since this modifies darius's listening events.)_

**How I found the root cause:**

1. Started at the feed route: `GET /feed/<user_id>/listening-now` in `routes/feed.py`, which calls `get_friends_listening_now()`.
2. In `services/feed_service.py`, the function builds `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD` (line 32) and keeps events where `listened_at >= cutoff` (line 42). So "recent" is defined entirely by that one constant.
3. Traced `RECENT_THRESHOLD` to the top of the module (line 13): `timedelta(hours=24)`. That was the mismatch — a feature literally called "listening **now**" was admitting anything from the last full day.
4. Cross-checked intent against the seed file: `seed_data.py:111` says recent events "within the past 30 minutes" should appear, and `:121` says older events "should NOT appear... after fix." So 24 hours was clearly wrong versus the documented ~30-minute intent.
5. **The moment of confidence** was the boundary control: moving darius's only listen from 10 h (shown) to 25 h (hidden) and changing *nothing else* flipped his inclusion. That proved the recency window was the cause — not the query, the friend lookup, or the per-friend dedup.

**Root cause:**

The "Friends Listening Now" feed selects listening events with `listened_at >= now - RECENT_THRESHOLD`, and `RECENT_THRESHOLD` was set to `timedelta(hours=24)`. A 24-hour window means a listen from the previous evening (e.g. 10 hours ago) still satisfies the filter the next morning, so a friend who played something at 11pm and hasn't opened the app since keeps appearing as if they're listening *right now* — until a full 24 hours elapses. The window was simply far too wide for a real-time/"today" feature; the intended window (per the seed comments) is about 30 minutes.

**Fix and side-effect check:**

Changed the recency window from 24 hours to 30 minutes:

```python
# services/feed_service.py:13
- RECENT_THRESHOLD = timedelta(hours=24)
+ RECENT_THRESHOLD = timedelta(minutes=30)
```

This fixes the root cause because the cutoff now excludes listens older than 30 minutes, so last night's activity no longer lingers into the next day. Side-effects checked afterward:
- **Bug case** — a listen 10 hours ago is now excluded (`False`).
- **No over-correction** — a listen 6 minutes ago still appears (`True`), so genuine current activity is preserved and the seed's 10–20-minute "recent" events still show.
- **Blast radius** — grepped `RECENT_THRESHOLD`: it's referenced only at its definition and inside `get_friends_listening_now`. The sibling `get_activity_feed` intentionally has no recency filter (per its docstring) and is unaffected, so the general activity feed still returns the most recent events regardless of age.

### Issue #5 — The last song in a playlist never shows up
**Reported by:** darius

Our collaborative playlist "Friday Energy" says it has 7 songs, but when I open it only 6 show. The missing one is always whatever was added most recently. Weirder: when simone added a new song, the previously-missing song suddenly appeared — and her new song became the missing one. So the playlist is always hiding exactly one song: the last one added.

**Steps I took:**

1. Opened the playlist (`GET /playlists/<playlist_id>/songs`) and counted the songs.
2. Added one more song (`POST /playlists/<playlist_id>/songs`) and re-fetched.

**Expected:** every song in the playlist is returned, including the newest.
**Actual:** the most recently added song is always missing; adding another song "frees" the previous one and hides the new one instead.

**How I reproduced it:** _(pending — not yet reproduced)_