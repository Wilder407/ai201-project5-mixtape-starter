## Codebase Map 

<!-- Cover at minimum: the main files and what each one does, the data flow for at least one feature (e.g., how sharing a song triggers a notification), and any patterns you notice in how the app is organized. -->

### Date Model Reference
- User, Song, Playlist, PlaylistSong (join table w/order column - explicit position, not inserrtion order), Notification
- No separate Rating model - rating lives directly on Song

### Entry Points
<!-- what calls into each service function? (routes in app.py, other services, background jobs). Note the file/line if you can. -->
**notification_service**
- `POST /songs/<id>/rate` -> `rate_song()`
- `POST /<playlist_id>/songs in routes/playlists.py` -> `add_to_playlist()`
- `GET /<user_id>/notifications` -> `get_notifications()`
- `POST /notifications/<notification_id>/read` -> `mark_as_read()`
- `create_notification()` has no direct route — called internally by other service functions

**playlist_service**
- `POST /` -> `create_playlist()`
- `GET /<playlist_id>` -> `get_playlist()`
- `GET /<playlist_id>/songs` -> `get_playlist_songs`

**search_service**
- `GET /song/search` -> `search_songs()`
- `GET /songs/<song_id>` -> `get_song()`

**streak_service**
- `POST /songs/<song_id>/listen` -> `record_listening_event()`
- `GET /<user_id>/streak` -> `get_streak`

**feed_service** 
- `GET /<user_id>/listening-now` -> `get_friends_listening_now`
- `GET /<user_id>/activity` -> `get_activity_feed`

### Pattern
- Routes = input parsing + response formatting only. Business logic lives entirely in `services/`. 
- Exceptions:   get     /users/<user_id> in users.py queries User directly via db.session.get, bypassing the service layer. Only route that does this - worth checking if intentional or an inconsistency. 

### Data Flow Trace 
<!-- pick one representative request and trace it end-to-end: route → service function(s) → models.py → back to response. This usually surfaces where behavior diverges from expectation. -->
1. `POST /songs/<id>/rate` hits `routes/songs.py`
2. -> `notify_song_rated()` in `notification_service.py`
3. -> creates Notification recod for song's original sharer
4. -> <!-- what doest he response look like? does the route return anything from notify_song_rated, or does it separately update Song.rating? -->

### Suspicious spots 
<!-- flag anything that looks off as you go: type mismatches, unclear error handling, hardcoded values, logic that contradicts what the README says should happen. Don't fix yet, just flag. -->

- `add_to_playlist -> create_notification(user_id=user_id)`: README says sharer should be notifies, but user_id appears to be the adding user vs sharer. Need to check where user_id is set before concluding this the a bug vs. just a misleading name. 

`get_user_playlists()` — check your work. Look at playlist_service.py: it defines create_playlist, get_playlist_songs, get_playlist, and get_user_playlists. But scan playlists.py's routes again — is there a route that calls get_user_playlists? If not, that's worth a note, not silence. A function with no entry point calling it is either: dead code, or a missing route (maybe one of your 3 bugs — a feature that should exist but the route was never wired up). Either way, flag it.

### How You Reproduced it 
5. The last song in the playlist never shows up. I ran tests.py and observed that the length of the test output was less than expected. I reviewed `playlist_seach.py` and looked for lines that generate output. The function calls `to_dict` from models.py but this only generates the dicts for the list. I further reviewed the for loop and noticed that the index [:-1] is used. Typically this makes sense to run through the end of the list. But, it appears that the loop is stopping before the end. 
1. Sunday
Reviewed tests.py and traced the code through the functions calls. Running tests.py confirmed that the assert 1 == 2 for adding 1 days to Sunday proves that the count is being reset. I further reviewed the paramenters and Sunday is indicated as weekday =6 with datetime method. Looking at streak_service I found an elif that indicated incrementing when the weekday != 6 then to set streak to 1. This is what is causing the reset bug. 

5. 
The bug shows up in the return statement of the get_playlist_songs function `return [song.to_dict() for song in songs[:-1]]`. The off-by-one error trunctates the for loop before adding the last song. The loop condition exits the for loop at the last song so this is never added to the playlist. The loop doesn't complete and call `to_dict` for the final song. Bug fixed by changing the index to the full list `[:]` instead of `[:-1]`.
3. The same song keeps showing up twice in search


