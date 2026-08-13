# Schema

Schema for usf class picker.

Search criteria (Campus, College, Department, Part of Term, Status filters,
Meets-on-days, Degree Program Attributes, etc. — everything on the Staff
Search form) are scraper *inputs* only and are never persisted. Only actual
result fields for a tracked class are stored, across three tables. See
`ARCHITECTURE.md` §4a for the search → track → poll flow these tables serve.

## tracked_classes

One row per tracked class. Denormalized on purpose — this is the only
persisted copy of a class's static info (see §4a), so display fields live
here directly rather than being joined from a search-results cache.
Soft-deleted via `untracked_at` rather than removed, so `poll_history` for a
class the user previously tracked stays intact.

| Field | Type | Constraints |
| --- | --- | --- |
| id | Integer | PK, autoincrement |
| crn | Number, 5 digits | not null |
| semester | Text (term code — format TBD, see `BACKLOG.md` Phase 1) | not null |
| prefix | Text | not null |
| number | Text | not null |
| title | Text | not null |
| credits | Number | not null |
| instructor | Text | nullable (TBA sections) |
| level | Text | nullable |
| instructional_method | Text | nullable |
| date_added | Timestamp | not null, default now |
| untracked_at | Timestamp | nullable (soft delete marker) |
| | | UNIQUE(crn, semester) |

## poll_history

One row per poll of a tracked class. This is the single source of truth for
a class's current status — "current status" is read as the latest row per
`tracked_class_id`, never duplicated onto `tracked_classes` itself.

| Field | Type | Constraints |
| --- | --- | --- |
| id | Integer | PK, autoincrement |
| tracked_class_id | Integer | FK → tracked_classes.id, not null |
| polled_at | Timestamp | not null |
| status | Text | not null |
| seats_available | Number | nullable (depends on what the results page exposes) |

## meeting_days

One row per meeting-day pattern for a tracked class (a section can meet on
multiple days, and lecture/lab components can differ — one row per day
keeps that a simple one-to-many instead of cramming a repeating group into
`tracked_classes` columns).

| Field | Type | Constraints |
| --- | --- | --- |
| id | Integer | PK, autoincrement |
| tracked_class_id | Integer | FK → tracked_classes.id, not null |
| day | Text (Mon–Sun) | not null |
| begin_time | Time | not null |
| end_time | Time | not null |
