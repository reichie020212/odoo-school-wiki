# `school_exams` — Module Blueprint

**Status:** approved (Red 2026-04-24: configurable scales, weighted course grade, ship a basic QWeb PDF).
**Depends on:** `school_academics`, `school_base`, `mail`.
**Scope:** assessment lifecycle — exam types, exams, per-student results, and per-term report cards with weighted grading and a printable PDF.

## Design decisions

1. **Grading scale** — configurable via `school.grading.scale` + `school.grading.scale.band`. One scale is marked default; each exam can override.
2. **Course grade formula** — weighted: `Σ(result.percentage × exam.weight) / Σ(exam.weight for exams the student has results in)`. Missing results are skipped (no penalty); a report card line with zero graded exams shows **Incomplete**.
3. **Report card PDF** — ships a basic QWeb template printable from the record action and via a stat button.

## Models

### `school.grading.scale`
| Field | Type | Notes |
|---|---|---|
| `name` | `Char` | E.g. "US Standard", "DepEd PH". |
| `code` | `Char` | Unique. |
| `is_default` | `Boolean` | Exactly one record may be default. |
| `band_ids` | `One2many('school.grading.scale.band','scale_id')` | |
| `active` | `Boolean` | |

Constraints: exactly one record with `is_default=True` (enforced in `@api.constrains`).

### `school.grading.scale.band`
| Field | Type | Notes |
|---|---|---|
| `scale_id` | `Many2one('school.grading.scale')` | Required, ondelete cascade. |
| `letter` | `Char` | Required. E.g. "A", "A-", "1.00". |
| `min_percentage` | `Float` | Required, 0-100 inclusive. |
| `gpa_value` | `Float` | For GPA calc. E.g. 4.0 for A, 1.00 for DepEd top grade. |
| `sequence` | `Integer` | Default 10. |

The lookup for a percentage finds the band with the largest `min_percentage` ≤ the percentage. If no band matches, returns the last band (failing).

### `school.exam.type`
| Field | Type | Notes |
|---|---|---|
| `name` | `Char` | E.g. "Quiz", "Midterm". |
| `code` | `Char` | Unique. |
| `default_weight` | `Float` | Percent contribution used as seed for exam.weight. |
| `sequence` | `Integer` | |

### `school.exam`
| Field | Type | Notes |
|---|---|---|
| `name` | `Char` | Computed + editable (e.g. "Algebra 1 Quiz #1 — 10A"). |
| `course_id` | `Many2one('school.course')` | Required. |
| `exam_type_id` | `Many2one('school.exam.type')` | Required. |
| `date` | `Date` | Required. Default today. |
| `max_score` | `Float` | Default 100. |
| `weight` | `Float` | Default seeded from exam_type_id.default_weight via onchange. |
| `grading_scale_id` | `Many2one('school.grading.scale')` | Optional override; falls back to default scale. |
| `state` | `Selection` | `draft`, `published`, `closed`. |
| `result_ids` | `One2many('school.exam.result','exam_id')` | |
| `result_count` | `Integer` | Computed. |
| `average_percentage` | `Float` | Computed. |

Inherits `mail.thread`, `mail.activity.mixin`.

**Actions:**
- `action_publish()` — draft → published; publishes the exam for result entry.
- `action_close()` — published → closed (no more edits).
- `action_reset_to_draft()`.
- `action_generate_result_placeholders()` — creates a blank `school.exam.result` row for each enrolled student with score=0 and state left to teacher.

### `school.exam.result`
| Field | Type | Notes |
|---|---|---|
| `exam_id` | `Many2one('school.exam')` | Required, ondelete cascade. |
| `student_id` | `Many2one('res.partner')` | Domain `[('is_student','=',True)]`, required. |
| `course_id` | `Many2one('school.course')` | Related from `exam_id.course_id`, stored. |
| `score` | `Float` | 0 ≤ score ≤ `exam_id.max_score`. Default 0. |
| `percentage` | `Float` | Computed, stored. |
| `letter_grade` | `Char` | Computed from exam's scale. |
| `gpa_value` | `Float` | Computed from exam's scale. |
| `remark` | `Char` | Optional. |

**Constraints:**
- `UNIQUE(exam_id, student_id)`.
- `@api.constrains` — student must be enrolled in `course_id`.
- `@api.constrains` — score ∈ [0, max_score].

### `school.report.card`
| Field | Type | Notes |
|---|---|---|
| `name` | `Char` | Computed ("Alice Garcia — 2026–2027 S1"). |
| `student_id` | `Many2one('res.partner')` | Required, `is_student=True`. |
| `term_id` | `Many2one('school.academic.term')` | Required. |
| `academic_year_id` | `Many2one` | Related. |
| `line_ids` | `One2many('school.report.card.line','report_card_id')` | |
| `overall_percentage` | `Float` | Computed — weighted mean of line percentages by line weight (default 1.0 per course). |
| `overall_gpa` | `Float` | Computed — mean of line `gpa_value`. |
| `overall_letter` | `Char` | Computed via default scale. |
| `state` | `Selection` | `draft`, `final`, `distributed`. |
| `teacher_remark` | `Text` | Overall remark. |
| `issue_date` | `Date` | Default today. |

**Constraints:**
- `UNIQUE(student_id, term_id)`.

**Actions:**
- `action_generate_lines()` — creates one `school.report.card.line` per course the student is enrolled in for `term_id` that doesn't already have a line.
- `action_finalize()` — draft → final. Recomputes all line grades from current results.
- `action_distribute()` — final → distributed.
- `action_print_report()` — wraps `report.report_action` to produce the PDF.

### `school.report.card.line`
| Field | Type | Notes |
|---|---|---|
| `report_card_id` | `Many2one('school.report.card')` | Required, ondelete cascade. |
| `course_id` | `Many2one('school.course')` | Required. |
| `subject_id` | `Many2one` | Related from `course_id.subject_id`, stored. |
| `teacher_id` | `Many2one` | Related from `course_id.teacher_id`, stored. |
| `percentage` | `Float` | Computed — weighted avg of student's results on course exams. |
| `letter_grade` | `Char` | Computed. |
| `gpa_value` | `Float` | Computed. |
| `is_incomplete` | `Boolean` | True if zero graded exams. |
| `teacher_remark` | `Char` | Editable. |

## Weighted formula

For a given student + course:

```
graded_results = student's results whose exam is published or closed AND has weight > 0
if not graded_results: percentage = 0.0, is_incomplete = True
else:
    percentage = Σ(r.percentage × r.exam.weight) / Σ(r.exam.weight)
letter_grade = scale.lookup(percentage)
gpa_value   = scale.band_for_letter(letter_grade).gpa_value
```

## QWeb PDF

- `report.report_school_report_card` — `ir.actions.report`, `report_type="qweb-pdf"`, `report_name="school_exams.report_school_report_card"`.
- Template: A4 page, school header (`res.company` name + logo), student block (name, admission #, classroom), term + academic year, lines table (subject | teacher | letter | %), overall GPA + letter footer, signatures line.

## Security

Uses `school_base` groups. ACL additions per model for Reader / User / Manager. Configuration menu items (exam types, grading scales) are `groups="school_base.group_school_manager"`.

## Menus

- **School → Academics → Exams** → `action_school_exam`
- **School → Academics → Exam Results** → `action_school_exam_result`
- **School → Academics → Report Cards** → `action_school_report_card`
- **School → Configuration → Exam Types** → `action_school_exam_type`
- **School → Configuration → Grading Scales** → `action_school_grading_scale`

## Demo data

- 2 scales — `scale_us` (A/B/C/D/F, 4.0 GPA) marked default, and `scale_deped_ph` (`1.00`/`1.25`/`1.50`/`1.75`/`2.00`/`2.25`/`2.50`/`2.75`/`3.00`/`5.00`).
- 3 exam types — `exam_type_quiz` (weight 20), `exam_type_midterm` (30), `exam_type_final` (50).
- Exams on `course_math_9a`:
  - `exam_math_9a_quiz1` — published, max 50, weight 20.
  - `exam_math_9a_midterm` — published, max 100, weight 30.
- Results for Alice (quiz 45/50, midterm 87/100) and Ethan (quiz 40/50, midterm 72/100).
- `report_card_alice_s1` — draft, 1 line for `course_math_9a`.

## Tests

- `test_grading_scale` — single default scale, band lookup, GPA mapping.
- `test_exam_type` — unique code, default weight preserved.
- `test_exam` — name autocomputes; state transitions; placeholder generation enrolls all students.
- `test_exam_result` — unique per (exam, student); enrollment enforcement; score range; percentage + letter + GPA.
- `test_report_card` — unique per (student, term); line generation enumerates student's courses; weighted percentage is correct; incomplete flag fires when no graded exams.

## Module layout

```
school_exams/
├── __init__.py
├── __manifest__.py
├── README.md
├── data/            (empty for now; could hold scales if seeded)
├── models/
│   ├── __init__.py
│   ├── school_grading_scale.py
│   ├── school_grading_scale_band.py
│   ├── school_exam_type.py
│   ├── school_exam.py
│   ├── school_exam_result.py
│   ├── school_report_card.py
│   └── school_report_card_line.py
├── report/
│   ├── __init__.py
│   ├── school_report_card_report.py (optional helper)
│   ├── school_report_card_templates.xml
│   └── school_report_card_actions.xml
├── security/
│   └── ir.model.access.csv
├── views/
│   ├── school_grading_scale_views.xml
│   ├── school_exam_type_views.xml
│   ├── school_exam_views.xml
│   ├── school_exam_result_views.xml
│   ├── school_report_card_views.xml
│   └── school_exams_menus.xml
├── demo/
│   ├── school_grading_scale_demo.xml
│   ├── school_exam_type_demo.xml
│   ├── school_exam_demo.xml
│   ├── school_exam_result_demo.xml
│   └── school_report_card_demo.xml
└── tests/
    ├── __init__.py
    ├── test_grading_scale.py
    ├── test_exam_type.py
    ├── test_exam.py
    ├── test_exam_result.py
    └── test_report_card.py
```
