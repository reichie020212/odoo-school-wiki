---
title: school_academics — Module Blueprint
description: Module blueprint for school_academics
published: true
date: 2026-04-28T05:42:46.029Z
tags: school_academics, blueprint, plan
editor: markdown
dateCreated: 2026-04-28T05:42:45.448Z
---

# `school_academics` — Module Blueprint

**Status:** draft, approved in principle (Red 2026-04-24: all 3 design decisions confirmed).
**Depends on:** `school_base`, `calendar`.
**Scope:** curriculum catalog, course offerings, session scheduling (via `calendar` RRULE), per-session attendance.

## Design decisions (Red, 2026-04-24)

1. **Auto-enrollment** — creating a course auto-enrolls the classroom's current roster. Explicit add/remove afterward is still allowed.
2. **Attendance granularity** — per-session, with a convenience "Mark all present" action on each session.
3. **Session generation** — pattern-based. Course carries a weekly schedule (day + time + duration); a "Materialize sessions" button generates `calendar.event` records across the term.

## Models

### `school.subject`
Curriculum catalog entry. Not a `product.template` — keeps academic and financial concerns separate (tuition billing will map subjects to products in a later `school_fees` module).

| Field | Type | Notes |
|---|---|---|
| `name` | `Char` | Required. E.g. "Algebra II". |
| `code` | `Char` | Required, unique. E.g. "MATH-10". |
| `description` | `Text` | Short description of the subject. |
| `default_weekly_hours` | `Float` | Default hours/week when generating courses. Default 3.0. |
| `active` | `Boolean` | Standard archive flag. |

Inherits `mail.thread`, `mail.activity.mixin`.

### `school.course`
Concrete offering of a subject for a given term, classroom, and teacher.

| Field | Type | Notes |
|---|---|---|
| `name` | `Char` | Computed + stored (e.g. "Algebra II — 10A — S1 2026–2027"). Editable. |
| `subject_id` | `Many2one('school.subject')` | Required. |
| `academic_year_id` | `Many2one('school.academic.year')` | Related from `term_id.academic_year_id`, stored. |
| `term_id` | `Many2one('school.academic.term')` | Required. |
| `teacher_id` | `Many2one('hr.employee')` | Domain `[('is_teacher','=',True)]`, required. |
| `classroom_id` | `Many2one('school.classroom')` | Required. |
| `student_ids` | `Many2many('res.partner')` | Domain `[('is_student','=',True)]`. Seeded from classroom roster on create. |
| `schedule_rule_ids` | `One2many('school.course.schedule.rule','course_id')` | Weekly meeting pattern. |
| `session_ids` | `One2many('calendar.event','school_course_id')` | Materialized sessions. |
| `session_count` | `Integer` | Computed, stored. |
| `state` | `Selection` | `draft`, `open`, `closed`. Default `draft`. |
| `color` | `Integer` | For kanban. |

Inherits `mail.thread`, `mail.activity.mixin`.

**Constraints:**
- Unique `(subject_id, classroom_id, term_id)` — one section of a subject per classroom per term. If electives show up later we'll relax this.

**Actions:**
- `action_materialize_sessions()` — reads `schedule_rule_ids`, iterates dates between `term_id.date_start` and `term_id.date_end`, creates `calendar.event` records with `school_course_id=self.id`. Skips dates that already have a session. Attendees = enrolled students' `res.partner`.
- `action_reset_enrollment()` — reseeds `student_ids` from `classroom_id.student_ids`.
- `action_open_sessions()` — opens the session list filtered for this course.
- `action_open_attendance()` — opens attendance filtered for this course.

**Overrides:**
- `create()` — seed `student_ids` from `classroom_id.student_ids` if not set in vals.

### `school.course.schedule.rule`
Weekly pattern for a course. One record per weekly meeting slot.

| Field | Type | Notes |
|---|---|---|
| `course_id` | `Many2one('school.course')` | Required, ondelete cascade. |
| `weekday` | `Selection` | `mon`, `tue`, `wed`, `thu`, `fri`, `sat`, `sun`. |
| `time_start` | `Float` | Widget `float_time`, e.g. 10.0 = 10:00. |
| `duration` | `Float` | Hours. Default from subject's `default_weekly_hours` / number of rules. |
| `location` | `Char` | Optional; defaults to classroom name in the materializer. |

**Constraints:**
- `time_start >= 0 and time_start < 24`.
- `duration > 0 and duration <= 8`.

### `calendar.event` (inherit)
Add school-context fields. No standalone `school.course.session` model — reuses Odoo's calendar entirely.

| Field | Type | Notes |
|---|---|---|
| `school_course_id` | `Many2one('school.course')` | Marks the event as a school session. |
| `school_attendance_ids` | `One2many('school.attendance','session_id')` | |
| `school_session_state` | `Selection` | `scheduled`, `completed`, `cancelled`. Default `scheduled`. Only meaningful when `school_course_id` is set. |
| `school_attendance_count` | `Integer` | Computed. |
| `school_present_count` | `Integer` | Computed. |

**Actions:**
- `action_mark_all_present()` — creates `school.attendance` rows (state `present`) for every enrolled student of `school_course_id` that doesn't already have a row, then flips `school_session_state` to `completed`.
- `action_open_attendance()` — opens the attendance list filtered for this session.

### `school.attendance`
Per-student, per-session record.

| Field | Type | Notes |
|---|---|---|
| `student_id` | `Many2one('res.partner')` | Domain `[('is_student','=',True)]`, required. |
| `session_id` | `Many2one('calendar.event')` | Domain `[('school_course_id','!=',False)]`, required, ondelete cascade. |
| `course_id` | `Many2one('school.course')` | Related from `session_id.school_course_id`, stored. |
| `state` | `Selection` | `present`, `absent`, `late`, `excused`. Default `present`. |
| `note` | `Char` | Optional. |
| `recorded_by_id` | `Many2one('res.users')` | Default `self.env.user`. |
| `recorded_at` | `Datetime` | Default now. |

**Constraints:**
- Unique `(student_id, session_id)`.
- Student must be enrolled in `session_id.school_course_id` (ORM-level check).

## Views

- `school.subject` — form/list/search/kanban.
- `school.course` — form (with stat buttons for sessions, attendance; notebook pages for students, schedule rules, sessions), list, search, kanban. "Materialize sessions" button in header.
- `calendar.event` form inherit — add "School Session" group shown when `school_course_id` is set, with session state, attendance count, "Mark all present" button.
- `school.attendance` — list (editable), form (rare; mostly edited inline), search.
- New menus: **School → Academics** submenu with *Subjects*, *Courses*, *Sessions* (calendar view filtered to school events), *Attendance*.

## Security

Reuses `school_base` groups (Reader / User / Manager):
- Reader → read-only on new models.
- User → read/write/create on subjects, courses (manage sessions, mark attendance).
- Manager → full CRUD including archive/state transitions.

ACL additions in `security/ir.model.access.csv`:
- `school.subject` × 3 groups.
- `school.course` × 3 groups.
- `school.course.schedule.rule` × 3 groups.
- `school.attendance` × 3 groups.
- (No new ACL on `calendar.event` — the inherited fields piggyback on calendar's existing ACLs.)

Record rules deferred.

## Demo data

- 4 subjects: `subject_math` (MATH-HS), `subject_science` (SCI-HS), `subject_english` (ENG-HS), `subject_history` (HIST-HS).
- 16 courses: each of the 4 classrooms (9A, 9B, 10A, 10B from `school_base`) × 4 subjects, with teachers distributed and a weekly schedule.
  - Example: `course_math_9a` — Subject Math, classroom 9A, teacher Maria Santos (homeroom), Mon/Wed/Fri 08:00–09:00 in "Room 9A".
- Materialize the first week of sessions to show the calendar integration.
- Attendance records for a single session to demonstrate.

## Tests

`tests/`:
- `test_subject.py` — CRUD, unique code, archive flag.
- `test_course.py` — create auto-enrolls classroom students; adding a student to classroom later does NOT auto-add to existing courses (Red can decide later); explicit add/remove works; unique (subject, classroom, term); `action_reset_enrollment` reseeds.
- `test_session_materialize.py` — a course with 2 schedule rules spanning a 2-week window creates the expected number of sessions; second call is idempotent (no duplicates).
- `test_attendance.py` — unique (student, session); `action_mark_all_present` creates rows for enrolled students only and flips session state; marking a student not enrolled raises.

## README / ERD

- `school_academics/README.md` — purpose, models, dependencies, how to materialize sessions, how attendance works.
- `docs/guide/school_academics-erd.md` — Mermaid ERD covering the 4 new models + `calendar.event` link + cross-references back to `school_base` models.
- `docs/guide/school_base-erd.md` — unchanged (no fields added to base models).

## Module layout

```
school_academics/
├── __init__.py
├── __manifest__.py
├── README.md
├── models/
│   ├── __init__.py
│   ├── school_subject.py
│   ├── school_course.py
│   ├── school_course_schedule_rule.py
│   ├── calendar_event.py
│   └── school_attendance.py
├── views/
│   ├── school_subject_views.xml
│   ├── school_course_views.xml
│   ├── school_course_schedule_rule_views.xml
│   ├── calendar_event_views.xml
│   ├── school_attendance_views.xml
│   └── school_academics_menus.xml
├── security/
│   └── ir.model.access.csv
├── demo/
│   ├── school_subject_demo.xml
│   ├── school_course_demo.xml
│   └── school_attendance_demo.xml
└── tests/
    ├── __init__.py
    ├── test_subject.py
    ├── test_course.py
    ├── test_session_materialize.py
    └── test_attendance.py
```
