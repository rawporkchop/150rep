# 150rep
ZotHacks Workout App with busy-ness meter for UCI students
Made by Anthony & Janice & Oliver & Al

Using SWIFT for the front end of the app, using SQL and Supabase for the database.

## Workout attendance calendar

The current API stores each visit as a `presence = TRUE` row in `user_data`,
using a time-zone-aware `timestamp`. It can return daily counts in the user's
IANA time zone; multiple visits on one day are counted separately. The
standalone [`workout_attendance.sql`](./database/schemas/workout_attendance.sql) file describes an
alternative table and is not used by this API.

## Workout session tables

To persist workout sessions, exercise logs, and sets in Supabase, run
[`workout_photos.sql`](./database/schemas/workout_photos.sql) first if the photo table and Storage
bucket are not already installed, then run
[`workout_sync.sql`](./database/schemas/workout_sync.sql) in the Supabase SQL editor. The migration
creates `workout_sessions`, `exercise_logs`, and `exercise_sets`, configures
per-user row-level security for exact `@uci.edu` addresses, and connects existing
photo metadata to sessions with a `NOT VALID` foreign key so legacy rows are not
rejected.

To store time-based exercises, run
[`workout_cardio.sql`](./database/schemas/workout_cardio.sql) after `workout_sync.sql`. It adds
`exercise_logs.is_cardio` (default `false`) and nullable
`exercise_sets.duration_seconds` (nonnegative integer, in seconds). It also
makes `weight` and `reps` nullable and removes their zero defaults, allowing
cardio sets to store duration without placeholder weight/repetition values.
Existing rows keep their current weight/reps values. The client must explicitly
write `duration_seconds` and leave weight/reps null for time-based sets; this
migration does not modify the Swift app or its sync payloads.

These tables are accessed directly through Supabase PostgREST with the signed-in
user's Supabase client; this migration does not add FastAPI routes or modify the
Swift app. After existing workouts and photos are synced, check for orphaned photo
rows before validating the foreign key:

```sql
SELECT p.id, p.session_id, p.created_at
FROM public.workout_photos AS p
WHERE NOT EXISTS (
    SELECT 1
    FROM public.workout_sessions AS s
    WHERE s.id = p.session_id
);

ALTER TABLE public.workout_photos
    VALIDATE CONSTRAINT workout_photos_session_id_fkey;
```

### Live workout visibility

Run [`active_workouts.sql`](./database/schemas/active_workouts.sql) in Supabase SQL Editor after
[`workout_sync.sql`](./database/schemas/workout_sync.sql) to add the session profile snapshot
fields and live heartbeat, and allow authenticated users to read active
workouts, their exercise logs, and sets while the heartbeat is newer than ten
minutes. The policies do not expose workout photos or permit watchers to write
to another user's workout. The migration does not change the Swift client; a
publisher must update `last_active_at` while a workout is open for it to appear
in the live feed.

## HTTP API for the Swift app

The Swift client calls this API over HTTPS; it must not connect directly to
PostgreSQL or contain database credentials. Signup and password login go through
the API, which delegates authentication to Supabase Auth and accepts only
`@uci.edu` email addresses. The API returns Supabase access and refresh tokens.
Protected API calls validate the access token and require its account email to
end in `@uci.edu`; the API creates a local `users` row on first request. Passwords
are handled by Supabase Auth and are not written to the app's database.

### Configure and run

1. Ensure the existing `users` table is created, then run
   [`api_migration.sql`](./database/migrations/api_migration.sql) in the Supabase SQL editor. It
   creates `user_data` and adds the Supabase-user link.
   The timestamp conversion assumes existing `user_data.timestamp` values were
   stored in UTC.
2. Copy [`.env.example`](./.env.example) to `.env` and set `SUPABASE_URL`,
   `SUPABASE_PUBLISHABLE_KEY`, and `DATABASE_URL` from your Supabase project.
   Keep `.env` private; only the publishable key is intended for client use.
3. Install dependencies and start the API:

   ```sh
   python -m pip install -r backend/requirements.txt
   cd backend && uvicorn api:app --host 0.0.0.0 --port 8000
   ```

Use the Supabase database connection string with SSL for `DATABASE_URL`. For a
deployed app, put the API behind HTTPS and configure these values as server-side
environment variables. Interactive API documentation is available at `/docs`.

### Endpoints

Signup and login do not require an existing access token:

```http
POST /v1/auth/signup
Content-Type: application/json
```

```json
{"email": "student@uci.edu", "password": "your-password"}
```

If Supabase email confirmation is enabled, the response has
`"confirmation_required": true`; the user must confirm their email, then log in.
Otherwise a session may be returned immediately.

```http
POST /v1/auth/login
Content-Type: application/json
```

Send the same JSON body to receive `access_token`, `refresh_token`, and user
details. Pass the access token to protected attendance and crowd-report routes:

```http
Authorization: Bearer <Supabase access token>
```

When the access token expires, refresh it through the API:

```http
POST /v1/auth/refresh
Content-Type: application/json
```

```json
{"refresh_token": "<Supabase refresh token>"}
```

This returns a new access token and refresh token. The API rejects non-UCI
emails for signup, login, refresh, and protected data routes.

Record a visit (the server uses the current time):

```http
POST /v1/attendance
Content-Type: application/json
```

To provide a visit time instead, send an ISO-8601 timestamp with an offset:

```json
{"attended_at": "2026-10-17T10:30:00-07:00"}
```

The response is `201 Created`:

```json
{"id": 123, "attended_at": "2026-10-17T17:30:00+00:00"}
```

Read daily counts for an inclusive range:

```http
GET /v1/attendance/calendar?start=2026-10-12&end=2026-10-18&timezone=America/Los_Angeles
```

The response includes every date in the range, including days with no visits:

```json
{
  "timezone": "America/Los_Angeles",
  "days": [
    {"date": "2026-10-12", "visits": 1},
    {"date": "2026-10-13", "visits": 0}
  ]
}
```

The date range is limited to 366 days. `GET /health` is a public process-health
check and does not verify database connectivity.

### Machine inventory and availability

After `api_migration.sql` has been run, run
[`machine_status.sql`](./database/schemas/machine_status.sql) in the Supabase SQL editor. It
creates:

- `public.machines` for the gym's machine inventory (`id`, `name`, optional
  `location`, and `created_at`).
- `public.machine_status` for each machine's latest shared status (`machine_id`,
  `is_in_use`, `updated_at`, and the local user who reported it).

The API connects to PostgreSQL using `DATABASE_URL`; these tables are accessed
through FastAPI, not directly through PostgREST. All three routes require a
valid bearer token for a Supabase account ending exactly in `@uci.edu`.
Any authenticated UCI account can add a machine or update its status. Status
updates are shared, last-write-wins reports and remain in effect until someone
updates them. Before the first report, `is_in_use` is `null` (unknown), not
`false`.

List the inventory and latest status:

```http
GET /v1/machines
Authorization: Bearer <Supabase access token>
```

```json
[
  {
    "id": 1,
    "name": "Leg Press 1",
    "location": "Weight room",
    "is_in_use": true,
    "updated_at": "2026-10-18T00:30:00+00:00"
  },
  {
    "id": 2,
    "name": "Cable Machine",
    "location": null,
    "is_in_use": null,
    "updated_at": null
  }
]
```

Add a machine to the inventory:

```http
POST /v1/machines
Authorization: Bearer <Supabase access token>
Content-Type: application/json
```

```json
{"name": "Leg Press 1", "location": "Weight room"}
```

Report whether a machine is in use (use the returned inventory `id`):

```http
PUT /v1/machines/1/status
Authorization: Bearer <Supabase access token>
Content-Type: application/json
```

```json
{"is_in_use": true}
```

The response contains `machine_id`, the saved `is_in_use` value, and
`updated_at`. A report for an unknown machine returns `404`.

### Crowd level

Run the updated [`api_migration.sql`](./database/migrations/api_migration.sql) in Supabase SQL Editor
to create the private `crowd_reports` table. Row-level security is enabled, so
the Swift app should call the API rather than querying this table through
PostgREST.

Submit or update the signed-in user's current crowd rating:

```http
POST /v1/crowd-reports
Authorization: Bearer <Supabase access token>
Content-Type: application/json
```

```json
{
  "level": 4,
  "latitude": 33.6487,
  "longitude": -117.8276,
  "accuracy_m": 12.5,
  "location_timestamp": "2026-10-18T03:05:00Z"
}
```

The API accepts a report only within 300 meters of the ARC center
(`33.6487, -117.8276` by default), with reported accuracy of at most 150 meters
and a timezone-aware location timestamp no more than 120 seconds old. Set
`ARC_LATITUDE` and `ARC_LONGITUDE` in the API environment to override the
center. Rejected locations return HTTP `422` (including `outside_geofence`,
`location_accuracy_too_low`, and `location_fix_expired`) without updating the
report or awarding karma. No new database table is needed: coordinates are
validated by the API and are not stored.

These required request fields change the API contract; existing app builds
that send only `level` will receive HTTP `422` until their client is updated.
No Swift client changes are included here. Device-reported coordinates can be
spoofed, so this checks reported proximity but cannot prove that a person is
physically inside the ARC.

Levels must be integers from 1 through 10. Each user has one current report;
submitting again updates that user's level instead of adding another vote. Each
accepted submission awards one karma point at most once per 30-minute window;
further submissions still update the current crowd report but do not add more
points in that window.

Before using karma, run [`karma_migration.sql`](./database/migrations/karma_migration.sql) in the
Supabase SQL editor after `api_migration.sql`. It adds the all-time karma counter,
the cooldown timestamp, and an optional avatar URL to `public.users`. The API
syncs the user's display name and HTTPS avatar URL from their verified Supabase
Auth profile metadata. Emails are not returned by the leaderboard.

Get the all-time leaderboard (default 50 entries, maximum 100):

```http
GET /v1/leaderboard?limit=50
Authorization: Bearer <Supabase access token>
```

```json
[
  {"rank": 1, "name": "Alex Example", "photo_url": "https://...", "karma": 12},
  {"rank": 2, "name": "UCI Student", "photo_url": null, "karma": 8}
]
```

The crowd-report response now also includes the submitter's total karma and
whether this submission earned a point:

```json
{
  "level": 6,
  "updated_at": "2026-10-18T02:30:00+00:00",
  "karma_points": 13,
  "karma_awarded": true
}
```

Get the average of the 10 most recently updated reports from the last 30
minutes:

```http
GET /v1/crowd-level
Authorization: Bearer <Supabase access token>
```

```json
{"level": 4.0, "report_count": 3}
```

`level` is the arithmetic mean and can be a decimal. Each user's latest report
replaces their previous vote. Only reports updated within the rolling last 30
minutes count, and at most the 10 newest are used. The API recalculates on each
GET request; no scheduled job is required. As reports age past 30 minutes, they
stop contributing. `report_count` is the number of users contributing to this
calculation (maximum 10), because each user has only one current report. With
no recent reports, the API falls back to the illustrative schedule for the
current hour in `America/Los_Angeles`. The stored 0–100 schedule value is
divided by 10 to match the live 1–10 scale, and `report_count` remains `0` to
show that the result came from historical rather than recent user reports.
Because the schedule only contains 6 AM–11 PM entries, `level` is `null`
outside those hours when there are no recent reports.

### Import the illustrative weekly chart

The workbook's “Page grid” is a separate illustrative 0–100 weekly schedule,
not live measurements and not 1–10 crowd votes. To load its 126 Monday–Sunday,
6 AM–11 PM values into Supabase, paste and run
[`seed_crowd_schedule.sql`](./database/seeds/seed_crowd_schedule.sql) in the Supabase SQL
Editor. It creates `public.crowd_schedule`, enables read-only access for
`anon` and `authenticated` roles, and safely updates matching schedule slots
when run again. `day_of_week` uses ISO numbering (Monday = 1, Sunday = 7);
`hour_of_day` uses 24-hour time. `is_illustrative` is always true for these
workbook values.

The Swift app can read the schedule from Supabase PostgREST using the project
URL and publishable key:

```http
GET https://<project-ref>.supabase.co/rest/v1/crowd_schedule?select=day_of_week,hour_of_day,level,is_illustrative&order=day_of_week,hour_of_day
apikey: <Supabase publishable key>
Authorization: Bearer <Supabase access token>
```

This returns rows like:

```json
[
  {"day_of_week": 1, "hour_of_day": 6, "level": 22, "is_illustrative": true},
  {"day_of_week": 1, "hour_of_day": 7, "level": 32, "is_illustrative": true}
]
```

The schedule is separate from `GET /v1/crowd-level`, which returns a rolling
average of recent user-submitted 1–10 votes.

Alternatively, fetch one day's 18 hourly values through the Python API:

```http
GET <API_BASE_URL>/v1/crowd-schedule?e=0
```

This schedule endpoint is public and does not require an access token. The `e`
parameter maps `0` to Sunday, `1` to Monday, continuing through `6` for
Saturday. It returns values divided by 10, ordered from 6 AM through 11 PM:

```json
{"e": 0, "values": [0.6, 1.0, 1.8, 3.0, 4.2, 5.0, 5.4, 4.8, 4.2, 3.8, 3.6, 3.4, 3.0, 2.6, 2.0, 1.2, 0.6, 0.3]}
```

For example, request `/v1/crowd-schedule?e=1` to get Monday's values. Run
[`seed_crowd_schedule.sql`](./database/seeds/seed_crowd_schedule.sql) in Supabase before using
this endpoint.

For UI testing, run [`seed_demo_crowd_reports.sql`](./database/seeds/seed_demo_crowd_reports.sql)
in a development/test Supabase project after running `api_migration.sql`. It
creates two synthetic users and repeatably sets their reports to 4 and 6, so the
response becomes `{"level": 5.0, "report_count": 2}` for 30 minutes after it is
run, if there are no other recent votes. Run it again to refresh the sample
timestamps. Do not run this in production: these sample reports are included
in the global average. To remove the sample rows, run:

```sql
DELETE FROM public.crowd_reports
WHERE user_id IN (
    SELECT id
    FROM public.users
    WHERE email IN (
        'demo-crowd-voter-1@example.invalid',
        'demo-crowd-voter-2@example.invalid'
    )
);

DELETE FROM public.users
WHERE email IN (
    'demo-crowd-voter-1@example.invalid',
    'demo-crowd-voter-2@example.invalid'
);
```

### Optional demo attendance data

To populate the current Los Angeles calendar week with four visits for visual
testing, run [`seed_demo_attendance.sql`](./database/seeds/seed_demo_attendance.sql) in the
Supabase SQL editor after replacing `REPLACE_WITH_TARGET_EMAIL` with the email
for an existing `users` row. The script adds only the visits needed to reach
four; it does not delete or label them as demo data. Prefer a dedicated test
account, and do not run it for a real user's account unless you intend to add
these records to their attendance history.
