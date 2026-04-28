# `school_base` — Module Blueprint

**Status:** Draft, pending Red's sign-off.
**Scope:** Foundation module for the School Management System. Provides the shared records (academic years, terms, classrooms, students, teachers, guardians) that every downstream module will depend on.
**Out of scope here:** courses/subjects, timetable, exams, fees, library, canteen, transport, portal, communications — those are separate modules layered on top.

## Design decisions (agreed with Red)

- **Students** — extend `res.partner` with an `is_student` boolean and student-specific fields (Odoo-idiomatic, same pattern as `customer_rank`/`supplier_rank`). No separate model.
- **Guardians / parents** — also `res.partner`, flagged with `is_guardian`. Linked to students via a symmetric m2m relationship (a student can have multiple guardians; a guardian can have multiple wards).
- **Teachers** — extend `hr.employee` with an `is_teacher` boolean and teaching-specific fields. Non-teaching staff (admin, janitors, cooks) live on the same `hr.employee` table without the flag.
- **Access control** — implemented with Odoo 19's `res.groups.privilege` / `res.groups` hierarchy (`privilege_id`, not `category_id`). Three groups: reader, user, manager.

## Module layout

```
school_base/
├── __init__.py
├── __manifest__.py
├── README.md
├── models/
│   ├── __init__.py
│   ├── res_partner.py
│   ├── hr_employee.py
│   ├── school_academic_year.py
│   ├── school_academic_term.py
│   └── school_classroom.py
├── views/
│   ├── res_partner_views.xml
│   ├── hr_employee_views.xml
│   ├── school_academic_year_views.xml
│   ├── school_academic_term_views.xml
│   ├── school_classroom_views.xml
│   └── school_menus.xml
├── security/
│   ├── school_security.xml
│   └── ir.model.access.csv
├── demo/
│   ├── school_academic_year_demo.xml
│   ├── school_classroom_demo.xml
│   ├── school_teacher_demo.xml
│   └── school_student_demo.xml
└── tests/
    ├── __init__.py
    ├── test_student_partner.py
    ├── test_teacher_employee.py
    ├── test_academic_year.py
    └── test_classroom.py
```

`depends`: `base`, `contacts`, `hr`, `mail`.

## Models

### 1. `res.partner` (inherit & extend)

New fields (all prefixed `student_` / `guardian_` for namespace hygiene, except the two flags):

| Field | Type | Notes |
|---|---|---|
| `is_student` | `Boolean` | Flags a partner as a student record. Default `False`. |
| `is_guardian` | `Boolean` | Flags a partner as a parent/guardian. Default `False`. Not mutually exclusive with `is_student` (adult learners with dependents). |
| `student_admission_number` | `Char` | Unique when `is_student=True`. Auto-generated via `ir.sequence` code `school.student.admission` on create if blank. Sequence prefix/padding editable from **School → Configuration → Admission Number Sequence**, giving UI control over the format (default: `%(range_year)s-`, padding 5, e.g. `2026-00001`). |
| `student_date_of_birth` | `Date` | |
| `student_gender` | `Selection` | `male`, `female`. |
| `student_admission_date` | `Date` | Default: today. |
| `student_status` | `Selection` | `enrolled`, `on_leave`, `graduated`, `withdrawn`. Default `enrolled`. |
| `student_classroom_id` | `Many2one('school.classroom')` | Current homeroom. |
| `student_academic_year_id` | `Many2one('school.academic.year')` | Denormalised helper — related on `student_classroom_id.academic_year_id`, stored. |
| `student_guardian_ids` | `Many2many('res.partner')` | Guardians for this student. Domain `[('is_guardian','=',True)]`. Uses a dedicated relation table `school_student_guardian_rel` with `student_id` / `guardian_id` columns. |
| `student_ward_ids` | `Many2many('res.partner')` | Inverse — students this partner is a guardian of. Same relation table, columns swapped. |
| `student_medical_notes` | `Text` | |
| `student_blood_type` | `Selection` | Standard A+/A-/B+/B-/AB+/AB-/O+/O-/unknown. |
| `student_emergency_contact_name` | `Char` | |
| `student_emergency_contact_phone` | `Char` | |

Constraints:
- `_sql_constraints` is deprecated in Odoo 19 — use `models.Constraint`:
  - Unique `student_admission_number` when not null.
- Python `@api.constrains`:
  - `student_date_of_birth` must be in the past.
  - A student cannot be their own guardian.

### 2. `hr.employee` (inherit & extend)

| Field | Type | Notes |
|---|---|---|
| `is_teacher` | `Boolean` | Flags an employee as a teacher. |
| `teacher_employee_number` | `Char` | School-side ID, distinct from HR's `barcode`. Unique when `is_teacher=True`. |
| `teacher_qualifications` | `Text` | Free-text summary (degrees, certifications). |
| `teacher_homeroom_classroom_id` | `Many2one('school.classroom')` | Homeroom assignment. Inverse of `school.classroom.homeroom_teacher_id`. |
| `teacher_hire_date` | `Date` | School hire date (distinct from HR contract start, which may differ). |

### 3. `school.academic.year`

| Field | Type | Notes |
|---|---|---|
| `name` | `Char` | e.g. `"2026–2027"`. Required. |
| `code` | `Char` | e.g. `"AY2627"`. Required, unique. |
| `date_start` | `Date` | Required. |
| `date_end` | `Date` | Required. |
| `state` | `Selection` | `draft`, `active`, `archived`. Default `draft`. |
| `is_current` | `Boolean` | Computed: `state == 'active'` and today between `date_start`/`date_end`. Searchable. |
| `term_ids` | `One2many('school.academic.term', 'academic_year_id')` | |

Constraints:
- `date_end > date_start`.
- At most one record with `state='active'` at a time (enforced via `@api.constrains` scanning siblings).

Lifecycle methods: `action_activate()`, `action_archive()`.

### 4. `school.academic.term`

| Field | Type | Notes |
|---|---|---|
| `name` | `Char` | e.g. `"First Semester"`. Required. |
| `code` | `Char` | e.g. `"S1"`. Required. |
| `academic_year_id` | `Many2one('school.academic.year')` | Required, ondelete `cascade`. |
| `date_start` | `Date` | Required. |
| `date_end` | `Date` | Required. |
| `sequence` | `Integer` | Default 10. |

Constraints:
- Term dates must fall within parent academic year's dates.
- Terms within the same academic year cannot overlap.
- `(academic_year_id, code)` unique.

### 5. `school.classroom`

| Field | Type | Notes |
|---|---|---|
| `name` | `Char` | Required. e.g. `"Grade 10 — Section A"`. |
| `code` | `Char` | Required, unique. e.g. `"10A"`. |
| `grade_level` | `Selection` | `k` (Kindergarten), `1`..`12`. Required. |
| `section` | `Char` | e.g. `"A"`. |
| `capacity` | `Integer` | Default 30. |
| `academic_year_id` | `Many2one('school.academic.year')` | Required. |
| `homeroom_teacher_id` | `Many2one('hr.employee')` | Domain `[('is_teacher','=',True)]`. |
| `student_ids` | `One2many('res.partner', 'student_classroom_id')` | With domain `[('is_student','=',True)]`. |
| `student_count` | `Integer` | Computed from `student_ids`, stored. |

Constraints:
- `student_count <= capacity` (warning via `@api.constrains`, allow override? → hard constraint for MVP).
- Unique `(academic_year_id, code)`.

## Views

- Per-model form / tree / search views with Odoo 19 syntax.
- `res.partner` form inherit: add a "School Information" notebook page visible only when `is_student=True`; add a "Guardians / Wards" page visible when `is_student` or `is_guardian`.
- `hr.employee` form inherit: add a "Teaching" notebook page visible when `is_teacher=True`.
- Search views use plain `<group>` for group-by filters (Odoo 19 dropped the `expand` attribute).
- Menus live under a top-level "School" root menu → Students, Teachers, Classrooms, Configuration → (Academic Years, Terms).
- Dedicated actions for students (`is_student=True` partners) and teachers (`is_teacher=True` employees) so clicks from the School menu only show the relevant records.

## Security

- Create `res.groups.privilege` record `school.privilege_management` with name "School Management".
- Three `res.groups` records, each linked via `privilege_id`:
  - `school.group_reader` — read-only on all school models and school-flagged partners/employees.
  - `school.group_user` — read/write on students, classrooms, terms; read on academic years.
  - `school.group_manager` — full CRUD including academic year lifecycle.
- `ir.model.access.csv` with one line per (model, group) combination.
- Record rules: teachers should only see students they teach (defer — requires course assignments from `school_academics`). For MVP, group-level ACLs only.
- No `category_id` anywhere on `res.groups` (Odoo 19 removed it).

## Demo data

- 1 academic year (2026–2027) with 2 terms (S1 Aug–Dec, S2 Jan–May).
- 4 classrooms (Grade 9A, 9B, 10A, 10B).
- 4 teachers with homeroom assignments, filed as `hr.employee`.
- 8 guardians (4 families × 2 parents).
- 8 students linked to those guardians, distributed across the classrooms.
- All XML files use `noupdate="1"` and external IDs referenceable by downstream modules (e.g. `school_base.student_alice_garcia`).

## Tests (`odoo.tests.common.TransactionCase`)

- `test_student_partner.py`:
  - Creating `res.partner` with `is_student=True` auto-generates admission number.
  - Admission number uniqueness enforced.
  - Guardian ↔ ward m2m is symmetric (assigning guardian shows up on the guardian's wards).
  - A student cannot be their own guardian.
- `test_teacher_employee.py`:
  - Creating `hr.employee` with `is_teacher=True` works and exposes teacher fields.
  - Non-teacher employee lookup excludes teachers and vice versa.
- `test_academic_year.py`:
  - Only one `active` year at a time.
  - `date_end > date_start` constraint.
  - Term dates must fall within academic year.
  - Overlapping terms within the same year rejected.
- `test_classroom.py`:
  - Classroom code unique per academic year.
  - `student_count` updates when students are assigned.
  - Homeroom teacher domain: cannot assign a non-teacher employee.

Run locally (requires a running Odoo db):
```bash
odoo-bin -d school_dev -i school_base --test-enable --stop-after-init
```

## ERD (`docs/guide/school_base-erd.md`)

Mermaid `erDiagram` covering:
- `res.partner` with `is_student` / `is_guardian` and student fields.
- `hr.employee` with `is_teacher` and teacher fields.
- `school.academic.year` → `school.academic.term` (1:N).
- `school.classroom` → `school.academic.year` (N:1), `hr.employee` (N:1 homeroom), `res.partner` (1:N students).
- `res.partner` ↔ `res.partner` (N:M guardian ↔ ward).

## README

Module root `README.md`. Sections: purpose, what it provides, dependencies, configuration, security groups, how to load demo data.

## Resolved with Red (2026-04-24)

1. **Grade levels:** K through 12 (`k`, `1`, `2`, …, `12`).
2. **Admission number format:** year-prefixed, via an `ir.sequence` whose prefix / padding are editable in the UI under **School → Configuration → Admission Number Sequence**. Default format: `2026-00001`.
3. **Gender:** `male` and `female` only.
4. **Homeroom teacher on classroom:** optional.

---
*This blueprint is a working artifact under `docs/plan/` — not committed. Once approved, implementation lands on `main` (Red opted out of worktree for this task).*
