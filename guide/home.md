---
title: Module ERDs
description: Per-module Entity Relationship Diagrams for the School Management System
published: true
date: 2026-04-28T07:57:45.542Z
tags: index, erd, schema
editor: markdown
dateCreated: 2026-04-28T07:57:45.030Z
---

# Module ERDs

Per-module Entity Relationship Diagrams. Each one shows the models, fields, and relationships that the module owns, plus how it plugs into the rest of the suite.

- [`school_base`](school_base-erd.md) — students, guardians, teachers, classrooms, academic years, terms.
- [`school_academics`](school_academics-erd.md) — subjects, courses, weekly schedules, sessions, attendance.
- [`school_exams`](school_exams-erd.md) — grading scales, exams, results, report cards.
- [`school_fees`](school_fees-erd.md) — fee items, structures, enrollments, scholarships, invoices.
- [`school_library`](school_library-erd.md) — book catalog, copies, loans, fines.
- [`school_canteen`](school_canteen-erd.md) — daily menus, prepaid wallets, allergy-aware purchases.
- [`school_shop`](school_shop-erd.md) — uniforms, textbooks, supplies (stock-backed).
- [`school_transport`](school_transport-erd.md) — buses, routes, stops, student assignments.
- [`school_comms`](school_comms-erd.md) — mail templates and automation rules.
- [`school_homepage`](school_homepage-erd.md) — public landing page at `/`.

For prose-level overviews of what each module does, see the [`reference/architecture`](../reference/architecture.md) page.
