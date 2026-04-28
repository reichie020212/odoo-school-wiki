---
title: school_library — Module Blueprint
description: Module blueprint for school_library
published: true
date: 2026-04-28T05:42:50.052Z
tags: school_library, blueprint, plan
editor: markdown
dateCreated: 2026-04-28T05:42:49.480Z
---

# `school_library` — Module Blueprint

**Status:** autonomous execution per Red's blanket approval 2026-04-24 (AFK).
**Depends on:** `school_base`, `mail`.
**Scope:** book catalog + physical-copy tracking + loan lifecycle with overdue detection and fine computation. No billing integration with `school_fees` for MVP (fines are tracked on the loan, admin can invoice manually).

## Decisions (autonomous, documented)

1. **Catalog split** — `school.library.book` (title metadata) separate from `school.library.book.copy` (physical copies). Loans attach to a copy, not a title, so 3-copies of the same book can circulate independently.
2. **Authors & categories** — dedicated `school.library.author` and `school.library.category` models (not `res.partner`). Keeps the library taxonomy clean and doesn't pollute the partner table.
3. **Borrowers** — any `res.partner` with `is_student=True` **or** a `hr.employee.user_partner_id` (for teachers). Kept flexible via domain.
4. **Fine calculation** — computed on the fly on `school.library.loan`:
   - `is_overdue = state in ('borrowed', 'returned_late') and due_date < today (or return_date > due_date)`.
   - `overdue_days` = `max(0, (min(return_date, today) − due_date).days)`.
   - `fine_amount` = `overdue_days × fine_per_day_rate`.
   - Rate is a config parameter `school_library.fine_per_day` (default 5.0) + default `loan_period_days` (default 14). Editable via Settings.
5. **Loan states** — `draft` → `borrowed` → `returned` | `returned_late` | `lost` . Automatic overdue flag + fine amount; admin explicitly marks `lost` to take the copy out of circulation.
6. **No fees integration** — for MVP, fines are data on the loan; no auto-invoicing. Future module can bridge to `school.fee.enrollment`.

## Models

| Model | Purpose |
|---|---|
| `school.library.category` | Genre/subject (Fiction, Science, History…). |
| `school.library.author` | Author (name, bio, optional photo). |
| `school.library.book` | Title: name, ISBN, `author_ids` m2m, `category_id`, publisher, publish_date, computed `copy_count`/`available_count`. |
| `school.library.book.copy` | Physical copy: `book_id`, `barcode` unique, `state` (`available` / `borrowed` / `lost` / `damaged`), `current_loan_id` helper. |
| `school.library.loan` | Borrow transaction: `book_copy_id`, `borrower_id`, `borrow_date`, `due_date`, `return_date`, `state`, computed `is_overdue` / `overdue_days` / `fine_amount`. |

### State transitions

- **Loan**: `draft` → **Confirm** → `borrowed` (sets copy state to `borrowed`, `due_date = borrow_date + loan_period_days`) → **Return** → `returned` (if on/before due_date) or `returned_late` (if after) → resets copy state to `available`. **Mark Lost** from `borrowed` sets copy to `lost` and loan to `lost`.
- **Book copy** state is driven by the active loan.

### Constraints

- `school.library.book.isbn` unique (when set).
- `school.library.book.copy.barcode` unique.
- A copy cannot be borrowed if its state is not `available` (`@api.constrains` on loan create).
- `due_date > borrow_date`.
- `return_date >= borrow_date` (when set).

## Views & menus

- **School → Library → Books / Copies / Loans**.
- **School → Configuration → Library Categories / Library Authors / Library Settings**.
- Book form shows the copies list with live availability. Loan form has **Confirm / Return / Mark Lost / Reset** buttons.

## Security

Uses `school_base` groups. Reader sees everything read-only. User creates loans and updates their state. Manager manages catalog and settings.

## Demo data

- 3 categories (Fiction, Science, History).
- 3 authors (J.K. Rowling, Carl Sagan, Howard Zinn).
- 3 books (Harry Potter, Cosmos, A People's History of the US), 2 copies each.
- 1 active loan — Alice Garcia borrowed Harry Potter copy #1, due today + 14 days.
- 1 returned loan — Ben Garcia returned Cosmos copy #1 on time.

## Tests

- `test_book_copy` — ISBN/barcode uniqueness, computed counts, lifecycle.
- `test_loan` — confirm sets copy to borrowed, return clears, fine = overdue_days × rate, cannot borrow unavailable copy, state machine.
- `test_config` — reading/setting the fine rate + loan period via `ir.config_parameter`.

## Module layout

```
school_library/
├── __init__.py
├── __manifest__.py
├── README.md
├── data/
│   └── ir_config_parameter_data.xml
├── models/
│   ├── __init__.py
│   ├── school_library_category.py
│   ├── school_library_author.py
│   ├── school_library_book.py
│   ├── school_library_book_copy.py
│   └── school_library_loan.py
├── security/
│   └── ir.model.access.csv
├── views/
│   ├── school_library_category_views.xml
│   ├── school_library_author_views.xml
│   ├── school_library_book_views.xml
│   ├── school_library_book_copy_views.xml
│   ├── school_library_loan_views.xml
│   └── school_library_menus.xml
├── demo/
│   ├── school_library_category_demo.xml
│   ├── school_library_author_demo.xml
│   ├── school_library_book_demo.xml
│   └── school_library_loan_demo.xml
└── tests/
    ├── __init__.py
    ├── test_book_copy.py
    └── test_loan.py
```
