# AI Usage

### Example 1: Issue #1

- I asked Claude how to reproduce a Sunday-only bug without waiting for Sunday. It suggested calling update_listening_streak() directly in the flask shell with hardcoded Saturday/Sunday datetimes.
- It verified that today.weekday() != 6 when I suspected it as the root cause and explained that Python's weekday() returns 6 for Sunday, causing it to be skipped.

### Example 2: Issue #4 

- I asked Claude to help me understand why add_to_playlist() worked but rate_song() didn't send a notification. It pointed me to compare the two functions side by side and that's when I saw that rate_song() didn't have any notification call.
- It gave me the code block to add, modeled on the pattern already used in add_to_playlist().
- I verified the fix by running the rating curl command, then checking the sharer's notifications endpoint and confirming the "song_rated" notification appeared with the correct body text.

### Example 3: Polishing submission.md

- I asked claude to help me format parts of my Root Cause Analyses especially when there were code snippets involved.
- I gave it a roughly written version of what I wanted to say and it added formatting to make it easier to look at.
  
# Codebase Map 

## Files and Responsibilities

**app.py** — Creates the app, configures SQLite, and registers the four route blueprints. The db object is also defined here and imported everywhere else.

**models.py** — Defines 7 database entities: User, Song, Playlist, Tag, ListeningEvent, Rating, and Notification. Also defines 3 tables: friendships , song_tags, and playlist_entries. All primary keys are UUIDs.

**routes/songs.py** — These are endpoints for sharing, searching, and rating songs. Calls search_service and notification_service.

**routes/playlists.py** — Endpoints for creating playlists and adding songs. Calls playlist_service and notification_service.

**routes/users.py** — Endpoints for user profiles, streak info, and notifications. Calls streak_service and notification_service.

**routes/feed.py** — Endpoints for the Friends Listening Now feed and activity feed. Calls feed_service.

**services/streak_service.py** — Tracks listening streaks. record_listening_event() creates a ListeningEvent and calls update_listening_streak(), which compares today's date to last_listened_at to decide whether to increment, hold, or reset the streak.

**services/feed_service.py** — get_friends_listening_now() queries ListeningEvent rows within the last 24 hours for a user's friends, and returns results. get_activity_feed() is similar but has no time cutoff just returns the latest events.

**services/search_service.py** — search_songs() does a case-insensitive ILIKE match on Song.title and Song.artist, with an outer join on song_tags to include tag data.

**services/notification_service.py** — add_to_playlist() adds a song to a playlist and fires a song_added_to_playlist notification to the original sharer. rate_song() saves a rating. get_notifications() retrieves a user's notifications.

**services/playlist_service.py** — get_playlist_songs() queries songs in a playlist ordered by position. create_playlist(), get_playlist(), and get_user_playlists() handles the rest of playlist management.

---

## Data Flow: User Rates a Song

1. Client sends POST /songs/<song_id>/rate with { "user_id": "...", "score": 4 }
2. routes/songs.py parses the body and calls notification_service.rate_song(user_id, song_id, score)
3. rate_song() validates the score (1–5), checks for an existing rating (updates it if found, creates a new Rating row if not), and commits to the DB
4. The route returns the rating as JSON

# Root Cause Analysis

## Issue #1: My listening streak keeps resetting

### 1. Issue number and title
Issue #1: My listening streak keeps resetting

### 2. How I reproduced it

Because the bug only triggers on Sundays, I reproduced it by calling the streak function directly in the Flask shell with simulated dates. I gave a user a streak of 5, set their `last_listened_at` to Saturday July 4, then called `update_listening_streak()` with a Sunday July 5 datetime so I could a user who listened on consecutive days across Saturday and Sunday:

```python
from datetime import datetime, timezone
from models import User
from services.streak_service import update_listening_streak

user = User.query.first()
user.listening_streak = 5
user.last_listened_at = datetime(2026, 7, 4, 12, 0, 0, tzinfo=timezone.utc)

update_listening_streak(user, datetime(2026, 7, 5, 12, 0, 0, tzinfo=timezone.utc))
print(user.listening_streak)  
```

The streak reset to 1 instead of incrementing to 6, confirming the bug.

### 3. How I found the root cause

The README identified `streak_service.py` as the affected file. I read `update_listening_streak()` and traced the logic. The function correctly computes `days_since_last` as the difference between today and the user's last listened date. The branch for incrementing the streak was:

```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
```

### 4. The root cause


The `today.weekday() != 6` is what we needed to change. Python's `weekday()` returns 0 for Monday through 6 for Sunday so this condition evaluates to false every Sunday, preventing the increment from ever running on that day. When both conditions aren't met, it jumps to the `else` branch, which resets the streak to 1.

### 5. My fix and side-effect check

Removed the `and today.weekday() != 6` as follows:

```python
elif days_since_last == 1:
    user.listening_streak += 1
```

This is correct because the streak rules want to increment on consecutive days, reset if a day is skipped and have nothing to do with which day of the week it is. 

After the fix I verified the original scenario showed up correctly and it did!

## Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

### 1. Issue number and title
Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

### 2. How I reproduced it

With the app running, I had nova rate a song shared by simone (Crown Heights Anthem), then checked simone's notifications:

```
# nova rates Crown Heights Anthem (shared by simone)
curl -X POST http://127.0.0.1:5000/songs/6e9759e2-36a7-45eb-bdf5-1fd345d4fb4b/rate \
  -H "Content-Type: application/json" \
  -d '{"user_id": "57ce89fc-f8e2-479d-bac7-55bb31a1acb8", "score": 5}'

# check simone's notifications
curl "http://127.0.0.1:5000/users/00728b03-18df-4f18-b567-4bc7a61e6bbb/notifications"
```

The rating returned a 201 response confirming it was saved, but simone's notification list came back empty (`"count": 0`).

### 3. How I found the root cause

The README identified `notification_service.py` as the affected file. I opened it and read both functions that handle user actions on songs: `add_to_playlist()` and `rate_song()`.

### 4. The root cause

`add_to_playlist()` performs the action (appending the song to the playlist), then calls `create_notification()` to alert the song's original sharer. Reading `rate_song()` side by side, I could see it saves the rating and then returns but with no call to `create_notification()` 

### 5. My fix and side-effect check

Added a `create_notification()` call at the end of `rate_song()`, with a safeguard to skip the notification if the rater is the song's own sharer:

```python
# Notify the song's sharer (if it wasn't them who rated it)
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```

After the fix, rerunning the same curl commands confirmed simone received a `"song_rated"` notification with the correct body. I also confirmed that `get_notifications()` and `mark_as_read()` were unaffected they operate on existing notification records and are independent of how notifications are created.

## Issue #5 — The last song in a playlist never shows up

### 1. Issue number and title
Issue #5: The last song in a playlist never shows up

### 2. How I reproduced it

With the app running, I fetched the songs for the "Late Night Vibes" playlist and counted the results:

```bash
curl http://127.0.0.1:5000/playlists/235220e0-6d2e-47e3-82d5-eba5c1df59c3/songs
```

The response contained 6 songs. I then verified the actual playlist contents in the Flask shell:

```python
from models import Playlist
p = Playlist.query.get("235220e0-6d2e-47e3-82d5-eba5c1df59c3")
print(len(p.songs), [s.title for s in p.songs])
# 7 ['Still Waters', 'Late Night Session', 'Free Throws', 'Golden Hour', 'First Light', 'Block Party', 'Midnight Drive']
```

The shell showed 7 songs but the API returned 6 "Free Throws" (the last song) was missing from the response.

### 3. How I found the root cause

The README identified `playlist_service.py` as the affected file. I opened `get_playlist_songs()` and read through it. The query correctly fetches all songs ordered by position ascending. The bug was in the return statement at the very end of the function:

```python
return [song.to_dict() for song in songs[:-1]]
```

### 4. The root cause

`get_playlist_songs()` correctly queries and orders all songs in any playlist, but the return statement applies a `[:-1]` conditionto the results list before returning it. This `list[:-1]` means it will return every element except the last one. This means no matter how many songs are in a playlist, the final song is always cut before the response is gotten. 

### 5. My fix and side-effect check

Removed the `[:-1]` slice from the return statement:

```python
# before
return [song.to_dict() for song in songs[:-1]]

# after
return [song.to_dict() for song in songs]
```

After the fix, the same curl command returned all 7 songs including "Free Throws." I also checked the other two playlists to confirm they returned their full song lists as well. The `create_playlist()`, `get_playlist()`, and `get_user_playlists()` functions in the same file were unaffected.
