---
title: Features
description: At-a-glance list of every feature the School Management System ships with
published: true
date: 2026-05-05T00:00:00.000Z
tags: features, overview, index
editor: markdown
dateCreated: 2026-05-05T00:00:00.000Z
---

# Features

A quick tour of everything the School Management System does, grouped by module. Each module name links to its ERD; cross-cutting capabilities at the bottom apply across the suite.

## 1. School Base — foundation

Covered by [`school_base`](guide/school_base-erd.md).

- Academic years and terms calendar
- Classroom and section management
- Student records (with guardian links)
- Guardian / parent profiles
- Teacher and staff profiles
- Demo personas for testing (student, guardian, teacher, registrar)

## 2. Academics

Covered by [`school_academics`](guide/school_academics-erd.md).

- Subject and course catalog
- Course offerings per term (subject × classroom × teacher)
- Automatic student enrollment into courses
- Weekly class schedule rules
- Auto-generated class sessions on the calendar
- Per-session attendance tracking

## 3. Exams & Grading

Covered by [`school_exams`](guide/school_exams-erd.md).

- Configurable grading scales (letter grades, GPA bands)
- Multiple exam types (quizzes, midterms, finals, etc.)
- Exam scheduling and student results entry
- Term report cards with weighted grade calculation
- Printable PDF report cards

## 4. Fees & Billing

Covered by [`school_fees`](guide/school_fees-erd.md).

- Fee items as catalog products (tuition, books, lab fees, etc.)
- Fee structures per grade level × term
- Per-student fee enrollments
- Scholarships (fixed-amount or percentage discounts)
- Invoice generation tied to accounting

## 5. Library

Covered by [`school_library`](guide/school_library-erd.md).

- Book catalog with per-copy tracking
- Loan checkout / return lifecycle
- Automatic overdue fine computation (configurable rate)

## 6. Canteen

Covered by [`school_canteen`](guide/school_canteen-erd.md).

- Menu items with dietary tags (vegetarian, halal, allergens, etc.)
- Student allergy profiles
- Prepaid meal wallets per student
- Top-ups and purchase transactions
- Allergy-based purchase blocks (safety)

## 7. School Shop

Covered by [`school_shop`](guide/school_shop-erd.md).

- Sale of uniforms, supplies, and merchandise
- Sales records with stock movements for physical goods

## 8. Transport

Covered by [`school_transport`](guide/school_transport-erd.md).

- Vehicle and driver management
- Bus routes with ordered stops
- Per-student route assignments (one active per academic year)

## 9. Parent / Student Portal

Lives at `/my/school` — see [Portal Guides](portal-guides/home).

- Self-service dashboard for students and guardians
- View report cards and exam results
- View fee enrollments and invoices
- Privacy rules ensuring users only see their own / ward data

## 10. Communications

Covered by [`school_comms`](guide/school_comms-erd.md).

- Email templates for key events
- Automated notifications: invoice issued, report card released, exam results published

## 11. Public Website / Homepage

Covered by [`school_homepage`](guide/school_homepage-erd.md).

- Public-facing landing page at `/`
- School identity and live statistics
- Calls-to-action for portal login and admin access
- Editable via Odoo's drag-and-drop website builder

---

## Cross-cutting capabilities

Inherited from Odoo 19 Community and applied across every module:

- Role-based access control (admin, registrar, teacher, student, guardian) — see [Permissions](reference/permissions.md)
- Activity logging and chatter on every record (notes, follow-ups, history)
- Document attachments (transcripts, ID copies, medical forms)
- Email integration for notifications
- Mobile-responsive backend and portal
- Multi-language support
- Reporting and dashboards
