# TuneLog Architecture & Design Decisions

This document outlines the technical challenges encountered during the development of TuneLog and the logic used to maintain data integrity.

---

## The "Ghost Flush" Mechanism

### The Problem: API Reporting Latency
TuneLog interacts with the Navidrome (Subsonic) API to monitor real-time listening activity. However, the Subsonic API has a known limitation: it does not actively report a **"Paused"** state. 

If a user pauses a track, the API continues to list that track as "Now Playing." If TuneLog simply calculated the difference between the `start_time` and the current `time.time()`, the "played duration" would continue to increase indefinitely reaching 100%, 200%, or even more while the song is actually paused.

### The Solution
To prevent these "phantom listens" from polluting the SQLite database and skewing recommendation scores, the watcher implements a **Ghost Flush**.

#### Logic Implementation:
```python
for entry in entries:
    user_id = entry["username"]
    song_id = entry["id"]
    
    if user_id in active and active[user_id]["song_id"] == song_id:
        # Calculate how long the song has been 'active' in our watcher
        elapsed = time.time() - active[user_id]["start_time"]

        # If elapsed time exceeds the actual song duration, the user has paused
        # or the API has failed to update the 'finished' status.
        if elapsed >= active[user_id]["duration"]:
            active.pop(user_id) # Remove from memory without DB commit
            print(f"[DONE] {user_id} finished: {entry['title']}")
            continue


