---
title: school_fees — Module Blueprint
description: Module blueprint for school_fees
published: true
date: 2026-04-28T05:42:49.259Z
tags: school_fees, blueprint, plan
editor: markdown
dateCreated: 2026-04-28T05:42:48.670Z
---

# `school_fees` — Module Blueprint

**Status:** approved (Red 2026-04-24, with explicit autonomy: all four design-question recommendations go).
**Depends on:** `school_base`, `account`, `product`, `mail`.
**Scope:** fee items → fee structures → per-student enrollments → customer invoices, with scholarships.

## Design decisions (all my recommendations, confirmed)

1. **Invoice partner** — the student (`res.partner` with `is_student=True`). Guardians added as followers so they get email visibility via chatter.
2. **Scholarship** — single record carries either a fixed amount **or** a percentage (XOR). Both cannot be set simultaneously (Python constraint).
3. **Invoice regeneration** — no in-place edit. If an enrollment changes after invoicing, admin cancels the invoice and regenerates.
4. **Fee structure scope** — tied to a `grade_level` (K–12) selection plus academic term. A Grade 9 structure automatically applies to every Grade 9 classroom.

## Models

### `product.template` (inherit)
Add `is_school_fee = fields.Boolean()` flag. Fee items are services with price + optional tax, fully reusing Odoo's product machinery.

### `school.fee.structure`
Bundle of fee items for a grade + term.

| Field | Type | Notes |
|---|---|---|
| `name` | `Char` | Computed from grade + term, editable. |
| `code` | `Char` | Unique. E.g. "FS-G9-S1-2627". |
| `grade_level` | `Selection` | K–12, required. |
| `term_id` | `Many2one('school.academic.term')` | Required. |
| `academic_year_id` | `Many2one` | Related from term. |
| `line_ids` | `One2many('school.fee.structure.line','structure_id')` | |
| `total_amount` | `Monetary` | Computed, stored. |
| `currency_id` | `Many2one('res.currency')` | Default from company. |
| `state` | `Selection` | `draft`, `open`, `closed`. |
| `active` | `Boolean` | |

Constraints: `UNIQUE(code)`, `UNIQUE(grade_level, term_id)`.

### `school.fee.structure.line`
| Field | Type | Notes |
|---|---|---|
| `structure_id` | `Many2one('school.fee.structure')` | Required, cascade. |
| `product_id` | `Many2one('product.product')` | Required, domain `[('is_school_fee','=',True)]`. |
| `name` | `Char` | Computed from product (editable override). |
| `quantity` | `Float` | Default 1. |
| `price_unit` | `Monetary` | Default from product.list_price. |
| `subtotal` | `Monetary` | Computed `qty × price`. |
| `sequence` | `Integer` | |

### `school.fee.enrollment`
Per-student fee enrollment.

| Field | Type | Notes |
|---|---|---|
| `name` | `Char` | Computed ("Alice Garcia — FS-G9-S1-2627"). |
| `student_id` | `Many2one('res.partner')` | `is_student=True`, required. |
| `structure_id` | `Many2one('school.fee.structure')` | Required. |
| `classroom_id` | `Many2one` | Related from `student_id.student_classroom_id`, stored. |
| `scholarship_ids` | `One2many('school.scholarship','enrollment_id')` | |
| `scholarship_total` | `Monetary` | Computed — sum of applied scholarships. |
| `amount_total` | `Monetary` | Computed — structure total − scholarship total. |
| `amount_paid` | `Monetary` | Computed from linked invoice. |
| `amount_due` | `Monetary` | Computed — `amount_total − amount_paid`. |
| `invoice_id` | `Many2one('account.move')` | Set when invoice generated; cleared when enrollment reset. |
| `payment_state` | `Selection` | Related from `invoice_id.payment_state`. |
| `state` | `Selection` | `draft`, `confirmed`, `invoiced`, `paid`, `cancelled`. |
| `term_id` | `Many2one` | Related from structure. |
| `currency_id` | `Many2one` | Related from structure. |

Constraints: `UNIQUE(student_id, structure_id)`. Also @api.constrains: student's classroom grade_level must match structure.grade_level.

Inherits `mail.thread`, `mail.activity.mixin`.

**Actions:**
- `action_confirm()` — draft → confirmed.
- `action_generate_invoice()` — confirmed → invoiced. Creates `account.move` (`move_type="out_invoice"`), `partner_id=student`, invoice_line_ids = structure lines + negative scholarship line, adds guardians as message followers.
- `action_cancel()` — cancels the linked invoice (if in draft/cancel state) and sets state to cancelled. If invoice is posted, raises UserError asking admin to reset the invoice to draft first.
- `action_reset_to_draft()` — only from cancelled, cleanly resets.

### `school.scholarship`
Discount against an enrollment.

| Field | Type | Notes |
|---|---|---|
| `name` | `Char` | Required. E.g. "Merit Scholarship — 10%". |
| `enrollment_id` | `Many2one('school.fee.enrollment')` | Required, cascade. |
| `discount_type` | `Selection` | `amount`, `percentage`, required. |
| `discount_amount` | `Monetary` | Used when type=amount. |
| `discount_percentage` | `Float` | Used when type=percentage; 0–100. |
| `computed_amount` | `Monetary` | Computed — the effective discount against the enrollment's structure total. |
| `reason` | `Text` | |
| `state` | `Selection` | `draft`, `applied`, `revoked`. |
| `currency_id` | `Many2one` | Related from enrollment. |

Constraints:
- If `discount_type=percentage` → 0 ≤ `discount_percentage` ≤ 100.
- If `discount_type=amount` → `discount_amount > 0`.
- A scholarship in `applied` state on an `invoiced` enrollment cannot be removed silently; must revoke first.

## Invoicing

- `account.move` creation: `move_type="out_invoice"`, `partner_id=student_id`, `invoice_date=today`, lines per `school.fee.structure.line` (name, quantity, price_unit, product_id), and one **negative** line per `applied` scholarship (`quantity=1, price_unit=-computed_amount, name="Scholarship: …"`).
- Invoice keeps chatter; guardians of the student are added as followers.
- We don't post automatically — the admin still reviews and posts the invoice in Accounting.

## Menus

- **School → Finance → Fee Enrollments** → `action_school_fee_enrollment`.
- **School → Finance → Scholarships** → `action_school_scholarship`.
- **School → Finance → Fee Invoices** → action filtered to invoices linked to enrollments.
- **School → Configuration → Fee Structures** → `action_school_fee_structure`.
- **School → Configuration → Fee Items** (manager) → filtered product list for `is_school_fee=True`.

## Security

Reuses `school_base` groups. Reader / User / Manager × each new model. Configuration entries (structures, fee items) are manager-only.

## Demo data

- 3 products: `product_tuition` (₱50,000), `product_lab_fee` (₱5,000), `product_activity_fee` (₱2,000).
- 2 structures:
  - `fee_structure_g9_s1` — Grade 9 S1 2026-2027 (tuition + lab + activity).
  - `fee_structure_g10_s1` — Grade 10 S1 2026-2027 (same items, slightly higher tuition).
- 2 enrollments (draft):
  - `enrollment_alice_g9_s1` — Alice Garcia (9A).
  - `enrollment_ben_g10_s1` — Ben Garcia (10A).
- 1 merit scholarship on Alice's enrollment — 10% percentage discount, applied state.

## Tests

- `test_product` — `is_school_fee` flag defaults False; promoted products show up in fee item filters.
- `test_fee_structure` — name autocompute, unique code, unique (grade_level, term), total rolls up from lines.
- `test_fee_enrollment` — unique (student, structure); `action_generate_invoice` creates proper `account.move`; amounts roll up correctly; scholarship deduction shows as negative line; enrollment state lifecycle; cancel flow.
- `test_scholarship` — XOR constraint on amount/percentage; computed_amount for both types; revoke path.

## Module layout

```
school_fees/
├── __init__.py
├── __manifest__.py
├── README.md
├── models/
│   ├── __init__.py
│   ├── product_template.py
│   ├── school_fee_structure.py
│   ├── school_fee_structure_line.py
│   ├── school_fee_enrollment.py
│   └── school_scholarship.py
├── views/
│   ├── product_template_views.xml
│   ├── school_fee_structure_views.xml
│   ├── school_fee_enrollment_views.xml
│   ├── school_scholarship_views.xml
│   └── school_fees_menus.xml
├── security/
│   └── ir.model.access.csv
├── demo/
│   ├── product_template_demo.xml
│   ├── school_fee_structure_demo.xml
│   ├── school_fee_enrollment_demo.xml
│   └── school_scholarship_demo.xml
└── tests/
    ├── __init__.py
    ├── test_product.py
    ├── test_fee_structure.py
    ├── test_fee_enrollment.py
    └── test_scholarship.py
```
