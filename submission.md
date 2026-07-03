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
