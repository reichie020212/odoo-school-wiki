---
title: School Management System — Home
description: Landing page for the odoo-school Wiki.js knowledge base
published: true
date: 2026-04-28T07:57:53.833Z
tags: overview, index, home
editor: markdown
dateCreated: 2026-04-28T07:35:00.019Z
---

# School Management System

A suite of **11 Odoo 19 Community Edition modules** covering the full student lifecycle: admissions, academics, exams, fees, library, canteen, shop, transport, parent portal, notifications, and a public homepage.

> **New here?** See the [Features →](features.md) page for an at-a-glance list of everything the suite does.

![Public homepage](_assets/02-homepage.png)

## What lives where

Each module name links to its ERD; the [Reference section](reference/home) covers cross-module concerns.

| Area | Covered by |
|---|---|
| Students, guardians, teachers, classrooms, academic years, terms | [`school_base`](guide/school_base-erd.md) |
| Subjects, courses, weekly schedules, sessions, attendance | [`school_academics`](guide/school_academics-erd.md) |
| Grading scales, exams, results, report cards + PDF | [`school_exams`](guide/school_exams-erd.md) |
| Fee items, structures, enrollments, scholarships, invoices | [`school_fees`](guide/school_fees-erd.md) |
| Book catalog, copy tracking, loans, overdue fines | [`school_library`](guide/school_library-erd.md) |
| Daily menus, prepaid wallets, allergy-aware purchases | [`school_canteen`](guide/school_canteen-erd.md) |
| Uniforms, textbooks, and supplies with stock integration | [`school_shop`](guide/school_shop-erd.md) |
| Buses, routes, stops, student transport assignments | [`school_transport`](guide/school_transport-erd.md) |
| Student/guardian dashboard at `/my/school` | `school_portal` (see [Portal Guides](portal-guides/home)) |
| Mail templates + automation rules | [`school_comms`](guide/school_comms-erd.md) |
| Public website-editable landing page at `/` | [`school_homepage`](guide/school_homepage-erd.md) |

## How to read this wiki

- **[Features →](features.md)** — full feature list grouped by module, plus the cross-cutting capabilities inherited from Odoo 19.
- **[Getting Started →](getting-started/home)** — install the stack, initialise a database, get to your first login.
- **[Admin Guides →](admin-guides/home)** — everyday workflows: adding students, creating exams, generating fee invoices, managing library loans, ...
- **[Portal Guides →](portal-guides/home)** — what students and guardians see when they sign in at `/my/school`.
- **[Reference →](reference/home)** — architecture diagrams, permission matrices, Odoo 19 specifics you need to know.
- **[Operations →](operations/home)** — backup, restore, upgrading between Odoo versions.

Every page that shows a UI flow has a screenshot captured from a fresh install with the shipped demo data — you should see the same thing on your install if you tick "Load demonstration data" when creating the database.

## Conventions

- Paths are relative to the repo root (e.g. `school_exams/models/school_exam.py`).
- Where a setting belongs in the Odoo UI, the full breadcrumb is spelled out: **Settings → Technical → Email → Templates**.
- Code blocks marked as `bash` run on your host (inside the project directory). Code blocks marked as `python` run inside the Odoo container shell.

## Support

Issues, questions, or feature requests: see the project repo's issue tracker. For operational emergencies, contact the school's IT administrator first — production data isn't fixable from a wiki.
