# GameHub — Understanding the system

10 questions to test your understanding of the data flow and architecture.
Work through them in order: read the code first, then run the app, then try to break things.

---

## How to investigate

You will need three things:

**1. Read the source code**
Start with `models.py` (the schema), then `seed.py` (the data), then `app.py` (the logic).
Many questions are answered entirely by reading carefully.

**2. Run the app and interact with it**
Use the UI at `http://localhost:5000` or send requests with curl or Postman.
Observe what actually happens — don't just reason about it.

```bash
# Example: log an activity for nova (id=1) on Hollow Knight (id=1)
curl -X POST http://localhost:5000/activities \
  -H "Content-Type: application/json" \
  -d '{"user_id": 1, "game_id": 1, "action": "started"}'
```

**3. Query the database directly**
Open `gamehub.db` with a SQLite tool and inspect the actual rows.

```bash
sqlite3 gamehub.db
.tables
SELECT COUNT(*) FROM notifications;
SELECT * FROM notifications WHERE user_id = 1;
```

Or use a GUI: **DB Browser for SQLite** (free, recommended).

---

## Suggested approach

| Phase | Questions | What you are doing |
|-------|-----------|-------------------|
| Read first | 1, 4, 8, 10 | Understand the code before touching anything |
| Then run it | 3, 6, 9    | Observe actual behaviour                     |
| Then break it | 2, 5, 7  | Try things, hit walls, reason about why      |

---

## Questions

**1.** When a user logs a new activity, how many database tables are written to?
List them and explain why each one is affected.

---

**2.** You call `DELETE FROM users WHERE id = 3` directly in SQLite.
What happens, and why? What would you need to do instead?

---

**3.** User `nova` changes her username to `nova_2`.
She then checks her friends' notification feeds.
What do they see — the old name or the new one? Why?

---

**4.** Trace the full journey of a `POST /activities` request.
Starting from the HTTP call, list every operation that happens before the response is returned.

---

**5.** `pixel_queen` opts out of activity tracking.
A teammate adds an `opted_out` boolean column to the `users` table and updates the `POST /activities` API route to check it.
Is the feature fully implemented? What did they miss?

---

**6.** How many rows are created in the database when `nova` logs one activity, given the current seed data?
Show your working.

---

**7.** You need to delete `maya_r`.
In what order must you delete rows across the tables, and why does the order matter?

---

**8.** The `notifications` table has a foreign key pointing to `activities`.
What happens if you try to delete an activity that has notifications attached to it?

---

**9.** A bug is found in the game catalog — wrong genre for one game.
You fix it and restart the app to ship the change.
What else just went down, and for how long?

---

**10.** A teammate says: *"let's just move the notification logic into its own function in `app.py`"*.
Does that solve the problem described in Task 4?
What is the actual architectural issue?

---

ANSWERS

1) Tables written when logging an activity

- activities
- notifications
- The app inserts one activity row, then one notification per friend.

2) `DELETE FROM users WHERE id = 3` run directly in SQLite

- It will fail if the user is still referenced by other rows.
- You must delete related notifications, activities, user_games, and friends first, or use cascading deletes.

3) `nova` renames to `nova_2` — what do friends see?

- The saved notification text still has the old name.
- The sender field uses the current user row, so that shows the new name.

4) Full journey of `POST /activities`

- Flask gets the request and parses JSON.
- The app opens the DB and checks the user.
- It inserts the activity.
- It fetches the activity and game.
- If notifications are on, it finds friends and inserts notifications.
- It commits and returns the activity.

5) `pixel_queen` opts out and teammate added `opted_out` + a check in `POST /activities`

- Not fully done.
- The API route checks opt-out, but the web form route still logs activities and the seed/schema need to support the flag.

6) Rows created when `nova` logs one activity (with current seed)

- 1 row in `activities`
- 2 rows in `notifications`
- Total = 3 rows.

7) Deleting `maya_r` — order and why

- Delete notifications she received.
- Delete notifications she triggered.
- Delete her activities.
- Delete her user_games.
- Delete her friendships.
- Delete her user row.
- The order matters because child rows must be removed before parent rows.

8) `notifications` references `activities`. What happens if you delete an activity with attached notifications?

- SQLite blocks it while foreign keys are on.
- Delete the notifications first or use `ON DELETE CASCADE`.

9) Fixing a game title and restarting the app — what else goes down and for how long?

- The whole app goes down during the restart.
- Users can’t use the service while it restarts.

10) Moving notification logic into its own function in `app.py` — does it solve Task 4?

- No.
- The issue is still that notifications are created inside the request, so the code is still coupled and blocking.

---

