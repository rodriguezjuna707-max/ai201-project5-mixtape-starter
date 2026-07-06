# Mixtape Codebase Map

This map explains how the app is built and how the pieces talk to each other. The goal is to understand the code well enough to know where each of the five issues lives. It does not try to fix anything.

---

## The big picture

Mixtape is a small Flask app backed by SQLite through SQLAlchemy. It follows one clear pattern from top to bottom:

**Request → Route → Service → Database**

Every route does three things and nothing more. It reads the incoming data. It calls one service function. It shapes the reply into JSON. All of the real thinking lives in the `services/` folder. So when a feature misbehaves you start at the route to see which service it calls, then you read that service.

There are three layers worth holding in your head:


| Layer | Folder      | Job                                    |
| ------- | ------------- | ---------------------------------------- |
| Data  | `models.py` | Defines the tables and how they relate |
| Logic | `services/` | Does the actual work                   |
| Web   | `routes/`   | Turns HTTP requests into service calls |

`app.py` wires everything together. It builds the Flask app, sets up the database, and registers the four route groups under their URL prefixes (`/songs`, `/playlists`, `/users`, `/feed`).

---

## The data model (`models.py`)

This file defines six tables and three association tables. Read it first because every service leans on it.

### The six main tables

**User** — a person on the app. Notable columns are `listening_streak` (an integer) and `last_listened_at` (a timestamp of their most recent listen). These two fields drive the whole streak feature. A user also has a `friends` relationship built on the `friendships` table.

**Song** — a shared track. Key column is `shared_by`, the id of the user who first shared it. This matters because notifications go to the sharer. A song also carries `shared_at` and a list of `tags`.

**Tag** — a genre label like "rap" or "lo-fi". A song can have many tags.

**ListeningEvent** — one row every time a user plays a song. It stores `user_id`, `song_id`, and `listened_at`. This is the raw material for both the streak logic and the "listening now" feed.

**Rating** — a user's score for a song from 1 to 5. There is no separate rating comment. A unique rule stops a user from rating the same song twice. Instead the score gets updated.

**Notification** — a message for a user. It has a `notification_type` string (for example `song_added_to_playlist` or `song_rated`) and a `body` of human text.

### The three association tables

These are the join tables that connect the main tables.

**friendships** links a user to a friend. The seed data inserts it in both directions so friendship is mutual.

**song_tags** links songs to tags.

**playlist_entries** is the important one. It joins a playlist to a song, but it also carries extra columns:

- `position` — an explicit integer for where the song sits in the playlist. Songs have a real order, not just the order they were inserted.
- `added_by` — who put the song there.
- `added_at` — when.

So a playlist is not just a bag of songs. Each entry knows its exact slot through `position`.

---

## The five services and how you reach them

Below is each service, what it is responsible for, and the exact call chain from a URL down to the function. This is the part that proves the reading. Follow one chain end to end and the rest read the same way.

### 1. `streak_service.py` — listening streaks

**What it does:** keeps a user's daily listening streak up to date.

The core function is `update_listening_streak(user, now)`. It compares today's date against the date in `last_listened_at` and decides what happens to the streak count. It has three cases. Same day means no change. One day gap means the streak grows. A bigger gap resets it to 1. There is also extra weekday logic inside the "one day gap" case that is worth reading closely.

**Call chain (recording a listen):**

```
POST /songs/<song_id>/listen
  → routes/songs.py  →  listen()
    → streak_service.record_listening_event(user_id, song_id)
      → creates a ListeningEvent
      → calls update_listening_streak(user, now)
```

**Call chain (reading a streak):**

```
GET /users/<user_id>/streak
  → routes/users.py  →  streak()
    → streak_service.get_streak(user_id)
```

**Files to understand this feature:** `streak_service.py`, plus the `listening_streak` and `last_listened_at` columns on **User** in `models.py`. The seed data in `seed_data.py` also sets `last_listened_at` for a few users, which is handy for testing.

---

### 2. `feed_service.py` — Friends Listening Now

**What it does:** shows which friends are listening right now.

The function `get_friends_listening_now(user_id)` builds a cutoff time from `RECENT_THRESHOLD` (set to 24 hours at the top of the file). It pulls the friend's `ListeningEvent` rows newer than that cutoff, sorts them newest first, then keeps only the most recent play per friend using a `seen_friends` set.

There is a second function in the same file, `get_activity_feed()`, which does something similar but is not filtered by time. It exists for contrast so read both to see the difference.

**Call chain:**

```
GET /feed/<user_id>/listening-now
  → routes/feed.py  →  listening_now()
    → feed_service.get_friends_listening_now(user_id)
      → reads the user's friends
      → queries recent ListeningEvent rows
```

**Files to understand this feature:** `feed_service.py`, the **ListeningEvent** and **User** models, and the `friends` relationship. The `RECENT_THRESHOLD` value at the top of the file controls what counts as "recent". The seed data plants both fresh and old listening events on purpose so you can see which ones show up.

---

### 3. `search_service.py` — song search

**What it does:** finds songs whose title or artist matches a query.

`search_songs(query)` runs one query. It joins `Song` to the `song_tags` table so tags come along, then filters where the title or artist contains the query text. It returns each matching song as a dict.

The detail to notice is how the join to `song_tags` interacts with songs that carry several tags. A song's tag count is set in `seed_data.py`, where some songs have zero tags, some have one, and some have three or more. That spread is deliberate.

**Call chain:**

```
GET /songs/search?q=<text>
  → routes/songs.py  →  search()
    → search_service.search_songs(query)
      → joins Song to song_tags, filters by title or artist
```

**Files to understand this feature:** `search_service.py`, the **Song**, **Tag**, and **song_tags** definitions in `models.py`, and the tag counts in `seed_data.py`.

---

### 4. `notification_service.py` — notifications

**What it does:** creates notifications when friends interact with your shared songs, and lets you read them.

This file holds several functions. The base helper is `create_notification(user_id, type, body)`, which every notification goes through. Two other functions represent the two kinds of interaction:

- `add_to_playlist(...)` adds a song to a playlist and then calls `create_notification` to tell the song's sharer.
- `rate_song(...)` saves a user's score for a song.

Read these two side by side. One of them builds a notification after doing its work and one of them does not. Comparing the two is the whole point.

**Call chain (rating a song):**

```
POST /songs/<song_id>/rate
  → routes/songs.py  →  rate()
    → notification_service.rate_song(user_id, song_id, score)
      → saves or updates a Rating
```

**Call chain (adding a song to a playlist):**

```
POST /playlists/<playlist_id>/songs
  → routes/playlists.py  →  add_song()
    → notification_service.add_to_playlist(playlist_id, song_id, added_by)
      → appends the song, then notifies song.shared_by
```

Notice that both write paths that touch songs run through this service, not through their "home" service. Song rating lives here rather than in a rating service, and playlist adding lives here rather than in the playlist service. That is because both actions produce notifications.

**Files to understand this feature:** `notification_service.py`, the **Notification**, **Rating**, and **Song** models. The `shared_by` column on **Song** decides who gets notified. The seed data creates one working `song_added_to_playlist` notification so you can see the correct shape.

---

### 5. `playlist_service.py` — playlist reading

**What it does:** creates playlists and returns their songs in order.

The key function is `get_playlist_songs(playlist_id)`. It queries `Song` joined to `playlist_entries`, filters to the one playlist, and orders the rows by `position` from low to high. Then it returns the list of song dicts. Read the very last line of that function closely, because it decides exactly which songs come back.

Note that creating a playlist and reading a playlist live here, but adding a song to a playlist does not. Adding lives in `notification_service.py` (see above) because it also fires a notification.

**Call chain:**

```
GET /playlists/<playlist_id>/songs
  → routes/playlists.py  →  get_songs()
    → playlist_service.get_playlist_songs(playlist_id)
      → joins Song to playlist_entries, orders by position
```

**Files to understand this feature:** `playlist_service.py`, and the **playlist_entries** join table with its `position` column. The seed data fills each playlist with several songs at known positions so you can count what comes back.

---

## How the issues map to files

Each issue points at one service, but understanding it always needs the model behind it. Here is the short version.


| # | Issue                                      | Read these files                                    |
| --- | -------------------------------------------- | ----------------------------------------------------- |
| 1 | Streak keeps resetting                     | `streak_service.py` + User model                    |
| 2 | Feed shows people from yesterday           | `feed_service.py` + ListeningEvent model            |
| 3 | Same song shows up twice in search         | `search_service.py` + song_tags join table          |
| 4 | Notified on playlist add but not on rating | `notification_service.py` + Notification model      |
| 5 | Last song in a playlist never shows        | `playlist_service.py` + playlist_entries join table |

---

## Where the test data comes from (`seed_data.py`)

This file is not part of the app logic, but it is your best friend for testing. It wipes the database and rebuilds it with five users, real friendships, twenty five songs with a mix of tag counts, three playlists at known positions, and listening events at known times. Reading it tells you exactly what data each feature should return, which makes it easy to spot when a feature returns the wrong thing.

The existing tests in `tests/` (`test_streaks.py`, `test_search.py`, `test_playlists.py`) show the expected behavior for three of the five features and are worth reading as living documentation.

---

# Bug Fixes

Each issue below uses the same five fields required by the root cause analysis format. The goal of each entry is to show that the bug was reproduced on purpose, tracked to its exact line, and fixed without breaking anything nearby.

- **Issue number and title** — the heading of each entry.
- **How I reproduced it** — the exact inputs, sequence of actions, or data state that triggers the behavior before any code was touched.
- **How I found the root cause** — which files I opened, my navigation path, and the moment I was sure I had the specific cause and not just a suspicious area.
- **The root cause** — in plain English, the exact condition, comparison, or missing step that caused the problem. It names the function and explains what it returns versus what the code assumed.
- **My fix and side-effect check** — what I changed, why that change fixes the root cause, and what related behavior I checked afterward to confirm nothing else broke.

---

## Issue #1 — My listening streak keeps resetting

**Affected service:** `streak_service.py`

**Status:** ✅ Fixed

**How I reproduced it:**
The bug only shows up on Sundays, so the trigger is a data condition, not a single request. I needed a user whose `last_listened_at` falls on a Saturday, then a new listen recorded on the Sunday right after. That is a one day gap, which by the rules should grow the streak.

Steps to confirm the bug before touching code:

1. Set a user's `last_listened_at` to a Saturday and give them a streak above 1 (for example a streak of 5).
2. Record a listen on the following Sunday. In the app this is `POST /songs/<song_id>/listen`, which runs `record_listening_event` and then `update_listening_streak`.
3. Watch the streak. It should climb to 6. Instead it drops to 1.

On any other day of the week the same one day gap grows the streak correctly. That is why the report said the reset felt random. It only happened once a week.

**How I found the root cause:**
I opened `streak_service.py` and followed the call path. `record_listening_event` creates the event then hands off to `update_listening_streak`, so the streak math lives in that second function. I read it top to bottom against its own docstring. The docstring lists four rules and none of them mention the day of the week. Then I reached this line:

```python
elif days_since_last == 1 and today.weekday() != 6:
```

The `today.weekday() != 6` part had no matching rule in the docstring. That was the moment I was sure. The symptom was "resets only sometimes" and this was the one branch whose behavior changed based on the calendar day. I confirmed it by recalling that `weekday()` counts Monday as 0, which makes 6 fall on Sunday. Symptom and code lined up exactly.

**The root cause:**
Python's `datetime.date.weekday()` returns 6 for Sunday. The increment branch required `days_since_last == 1 and today.weekday() != 6`. So on a Sunday the second half of the `and` is `False`, the whole branch is skipped, and control falls into the `else` that sets the streak back to 1. In plain terms, a listen made on a Sunday after listening the day before was treated as a broken streak instead of a one day gap, even though the gap was exactly one day. The extra weekday condition was never part of the streak rules, so it reset streaks every Sunday for no valid reason.

**My fix and side-effect check:**
I removed the weekday condition so the branch only checks the day gap.

```python
elif days_since_last == 1:
    user.listening_streak += 1
```

This fixes the root cause because the increment now depends only on whether the last listen was one day ago, which is what the docstring specifies. The day of the week no longer changes the outcome.

Side-effect check. I confirmed the other three branches still behave as documented: a first time listener with `last_listened_at is None` still starts at 1, a second listen on the same day still leaves the streak unchanged, and a gap larger than one day still resets to 1. I also confirmed `record_listening_event` still commits the new event and the streak change together. Then I ran the streak test suite with the project virtual environment:

```
& ".\.venv\Scripts\python.exe" -m pytest tests/test_streaks.py -q
5 passed
```

All five streak tests pass, including the Sunday boundary case. The search and playlist suites were left untouched by this change.

---

## Issue #2 — Friends Listening Now shows people from yesterday

**Affected service:** `feed_service.py`

**Status:** ✅ Fixed

**How I reproduced it:**
The trigger is a data condition about how old a listening event is. I set up a user with one friend, then gave that friend two listening events. One event was from earlier today and one was from two days ago.

Steps to confirm the bug before touching code:

1. Create a user and a friend, with friendship recorded between them.
2. Add one `ListeningEvent` for the friend dated two days ago and one dated earlier today.
3. Call `GET /feed/<user_id>/listening-now`, which runs `get_friends_listening_now`.
4. The feed returned both songs. The two day old event should not be in a "listening now" feed at all.

The seed data in `seed_data.py` shows the same shape on purpose. It plants fresh events from the last several minutes and stale events from hours and days ago, with a comment saying the stale ones should not appear.

**How I found the root cause:**
I opened `feed_service.py` and read `get_friends_listening_now` from top to bottom. The query filters events with `ListeningEvent.listened_at >= cutoff`, so the only thing deciding how far back the feed reaches is `cutoff`. I traced `cutoff` up one line to where it is built, and that is built from `RECENT_THRESHOLD` at the top of the file. That constant was set to `timedelta(hours=24)`. That was the moment I was sure. A 24 hour window means an event from yesterday is still inside the cutoff, which matches the report of "people from yesterday" exactly. The dedupe loop and the query were both fine. The single wrong value was the threshold.

**The root cause:**
`cutoff` was defined as `datetime.now(timezone.utc) - RECENT_THRESHOLD` with `RECENT_THRESHOLD = timedelta(hours=24)`. So the filter kept every listening event from the past 24 rolling hours. A friend who listened yesterday afternoon was still newer than a cutoff set to 24 hours ago, so they passed the filter and appeared in the feed. In plain terms the feed was treating "listened in the last day" as "listening now", so stale listens from yesterday leaked in.

**My fix and side-effect check:**
I changed the cutoff to the start of the current day in UTC, so only today's listens count. I removed the unused `RECENT_THRESHOLD` constant and the now unused `timedelta` import.

```python
# Start of the current day in UTC, so only today's listens count as "now"
cutoff = datetime.now(timezone.utc).replace(hour=0, minute=0, second=0, microsecond=0)
```

This fixes the root cause because a listen from yesterday now falls before midnight of today, so it is excluded by the `listened_at >= cutoff` filter.

Note on the tradeoff. A calendar day cutoff is a deliberate choice. It still shows a friend who listened at 2am today even if it is now late at night, which is wider than a strict real time window. I chose the calendar day boundary on purpose for simplicity and to match the "shows people from yesterday" complaint directly.

Side-effect check. There is no `test_feed.py` in the suite, so I wrote a small standalone check. It builds a user with a friend, adds one listen from earlier today and one from two days ago, then calls `get_friends_listening_now`. The result contained only today's song, and the two day old event was excluded. I also ran the full suite to confirm the change broke nothing:

```
& ".\.venv\Scripts\python.exe" -m pytest tests/ -q
2 failed, 11 passed
```

The two failures are both in `test_playlists.py` and belong to the still open Issue #5 (the test itself is marked `# Bug causes this to return 4`). The streak and search suites still pass, so the feed change had no side effects on them.

---

## Issue #3 — The same song keeps showing up twice in search

**Affected service:** `search_service.py`

**Status:** ✅ Fixed

**How I reproduced it:**
The trigger is a data condition about how many tags a song has. The seed data gives some songs zero tags, some one tag, and some three or more tags. That spread is what makes the bug show up for some songs and not others.

Steps to confirm the bug before touching code:

1. Search for a term that matches a song with three or more tags. In the app this is `GET /songs/search?q=<text>`, which runs `search_songs`.
2. Look at the results. The song came back three times, once per tag.
3. Search for a term that matches a song with one tag. That song came back only once and looked fine.

The count of duplicates matched the tag count exactly, which is why the report felt inconsistent. A one tag song looked correct while a three tag song was tripled.

**How I found the root cause:**
I opened `search_service.py` and read `search_songs`. The query does three things: it joins `Song` to `song_tags`, it filters on title or artist, and it returns each row as a dict. I checked the filter first and it only touches `Song.title` and `Song.artist`, so the filter was not the problem. That pointed me at the join. The join condition is `Song.id == song_tags.c.song_id`, and `song_tags` holds one row per song and tag pair. So a three tag song produces three joined rows. I confirmed nothing after the join collapsed those rows back down. That was the moment I was sure. The duplication was coming from the join fanning out rows, not from bad data.

**The root cause:**
`search_songs` called `.outerjoin(song_tags, Song.id == song_tags.c.song_id)`. The `song_tags` table stores one row per song and tag pair, so joining to it produces one result row per tag. A song with three tags became three rows, and the query never called `.distinct()` or grouped the rows, so `.all()` returned the same song three times and the final list held it three times. In plain terms the query multiplied each song by its number of tags. The join was not even needed, because the filter uses only title and artist and the tags are loaded separately inside `song.to_dict()`.

**My fix and side-effect check:**
I removed the join to `song_tags` entirely, since nothing in the query used it. I also removed the now unused `Tag` and `song_tags` imports.

```python
# No join to song_tags: the filter only uses title and artist, and tags
# are loaded separately inside to_dict(). Joining here would return a song
# once per tag, which is what caused duplicate search results.
results = (
    db.session.query(Song)
    .filter(
        db.or_(
            Song.title.ilike(f"%{query}%"),
            Song.artist.ilike(f"%{query}%"),
        )
    )
    .all()
)
```

This fixes the root cause because with no join there is one row per song, so a song appears once no matter how many tags it has. The tags still show up in the result because `to_dict()` reads them from the `Song.tags` relationship on its own.

Side-effect check. I confirmed the tags still appear in each result dict by reading `to_dict()`, which builds the tag list from the relationship rather than from the query. Then I ran the search suite:

```
& ".\.venv\Scripts\python.exe" -m pytest tests/test_search.py -v
5 passed
```

All five search tests pass, including the three that specifically check for no duplicates on zero tag, one tag, and multi tag songs. I also ran the full suite. The only failures are the two in `test_playlists.py` that belong to the still open Issue #5. The streak suite still passes, so the search change had no side effects on the rest of the app.

---

## Issue #4 — Notified when a friend added my song to a playlist but not when they rated it

**Affected service:** `notification_service.py`

**Status:** ✅ Fixed

**How I reproduced it:**
The trigger is a specific sequence of actions between two users. One user shares a song and a second user rates it. The sharer should get a notification but does not.

Steps to confirm the bug before touching code:

1. Create a sharer and a rater, both users.
2. Have the sharer share a song, so `song.shared_by` is the sharer.
3. Have the rater rate that song. In the app this is `POST /songs/<song_id>/rate`, which runs `rate_song`.
4. Check the sharer's notifications with `GET /users/<user_id>/notifications`. The list was empty.

For contrast I ran the same shape with a playlist add instead of a rating, and that one did create a notification. So the same interaction produced a notification in one path and nothing in the other.

**How I found the root cause:**
I opened `notification_service.py` and compared the two functions that handle a friend touching your song. `add_to_playlist` ends with a clear pattern. After it does its work it checks `if song.shared_by != added_by_user_id` and then calls `create_notification` to tell the sharer. I then read `rate_song` top to bottom looking for the same pattern. It validates the score, loads the song and the rater, saves or updates the `Rating`, commits, and returns. There was no call to `create_notification` anywhere in it. That was the moment I was sure. The working path and the broken path sat side by side, and only one of them notified.

**The root cause:**
`rate_song` saved the rating but never called `create_notification`, so no notification row was ever created for a rating. The sibling function `add_to_playlist` did call `create_notification` after its work, which is why playlist adds notified the sharer and ratings did not. In plain terms the notify step was simply missing from `rate_song`. The rating was stored correctly, but nobody was told about it.

**My fix and side-effect check:**
I added the missing notification to `rate_song`, mirroring the exact pattern `add_to_playlist` already uses. It runs after the commit, notifies `song.shared_by`, and skips the case where a user rates their own song.

```python
db.session.commit()

# Notify the person who originally shared the song (if it wasn't them who rated it)
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score} out of 5.",
    )

return rating
```

This fixes the root cause because rating a song now creates a `song_rated` notification for the sharer, matching how playlist adds already behave. The `song_rated` type matches the one named in the `Notification` model docstring and the codebase map.

Side-effect check. There is no `test_notifications.py` in the suite, so I wrote a small standalone check. It creates a sharer and a rater, shares a song, then has the rater rate it. The sharer received exactly one notification of type `song_rated` with the expected body. I also checked the self rating case where the sharer rates their own song, and no notification was created, so the guard works. The rating itself still saves and updates as before, and `add_to_playlist` was left untouched. I ran the full suite to confirm nothing else broke:

```
& ".\.venv\Scripts\python.exe" -m pytest tests/ -q
2 failed, 11 passed
```

The two failures are both in `test_playlists.py` and belong to the still open Issue #5. The streak and search suites still pass, so the notification change had no side effects on them.

---

## Issue #5 — The last song in a playlist never shows up

**Affected service:** `playlist_service.py`

**Status:** ✅ Fixed

**How I reproduced it:**
The trigger is any playlist that has songs in it. The last song by position is always the one that goes missing.

Steps to confirm the bug before touching code:

1. Take a playlist with a known number of songs. The seed data builds playlists with several songs at set positions, and the test fixture builds one with five.
2. Read the playlist's songs. In the app this is `GET /playlists/<playlist_id>/songs`, which runs `get_playlist_songs`.
3. Count the results. A five song playlist returned only four songs, and the missing one was `Track 5`, the last by position.

The two playlist tests show this directly. `test_playlist_returns_all_songs` expected 5 and got 4, and `test_playlist_returns_songs_in_order` reported the result was missing the final `Track 5`.

**How I found the root cause:**
I opened `playlist_service.py` and read `get_playlist_songs`. The query itself looked correct. It joins `Song` to `playlist_entries`, filters to the one playlist, and orders by `position` ascending, so `songs` holds the full ordered list. I then read the return line, which was:

```python
return [song.to_dict() for song in songs[:-1]]
```

The `[:-1]` slice was the moment I was sure. It means "every item except the last one," so the function was throwing away the final song right before returning. The query fetched all of them and the slice dropped one on the way out. The docstring above even says the function "returns all songs in the playlist," which the code contradicted.

**The root cause:**
The return statement sliced the result list with `songs[:-1]`. In Python that slice returns every element except the last, so the highest position song was always removed before the list was returned. The database query was correct and fetched every song. The bug was purely in the return line dropping the tail item. That is why the last song in a playlist never showed up while every earlier song was fine.

**My fix and side-effect check:**
I removed the slice so the function returns every song the query found.

```python
return [song.to_dict() for song in songs]
```

This fixes the root cause because there is no longer any step that drops an element. Every song from the ordered query is returned, so the last song by position now appears. I chose plain `songs` over `songs[:]` on purpose. Both return all songs, but `songs[:]` looks like a deliberate copy and would leave a reader wondering why, which is the same kind of confusion that hid the original `[:-1]`.

Side-effect check. I confirmed the ordering is untouched, since the fix only removes the slice and leaves the `order_by(position)` query in place, so songs still come back in position order. I also confirmed the empty playlist case still works, because slicing an empty list and returning an empty list give the same result. Then I ran the playlist suite and the full suite:

```
& ".\.venv\Scripts\python.exe" -m pytest tests/test_playlists.py -v
3 passed

& ".\.venv\Scripts\python.exe" -m pytest tests/ -q
13 passed
```

All three playlist tests pass now, including the two that were failing before. The full suite is green at 13 passed with no failures, so this fix broke nothing elsewhere.
