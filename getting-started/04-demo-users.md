---
title: Demo users
description: Pre-seeded demo accounts for trying out each role in the School Management System
published: true
date: 2026-05-05T00:00:00.000Z
tags: getting-started, demo, users, login, credentials
editor: markdown
dateCreated: 2026-05-05T00:00:00.000Z
---

# Demo users

When you install the suite with demo data (the default for a fresh install, i.e. `--without-demo=False` was *not* passed), the database is seeded with one ready-to-use account per major role. Use these to try out the system from each user's perspective without having to create users yourself.

> Login page: **<http://localhost:8069/web/login>** (backend) or **<http://localhost:8069/my/school>** (portal).

## Accounts

| Role | Login | Password | Where they land |
|---|---|---|---|
| **Student** | `demo_student` | `demo_student` | Portal at `/my/school` — sees only their own grades, fees, schedule, report cards. |
| **Guardian** | `demo_guardian` | `demo_guardian` | Portal at `/my/school` — sees data for the student(s) linked to this guardian account. |
| **Teacher** | `demo_teacher` | `demo_teacher` | Backend at `/web` — can mark attendance, enter exam results, and add report-card comments for the classes they teach. |
| **Registrar** | `demo_registrar` | `demo_registrar` | Backend at `/web` — full administrative access: students, classrooms, exams, fees, library, canteen, shop, transport. |
| **Program Admin** | `demo_admin` | `demo_admin` | Backend at `/web` — can do anything across the suite **except** installing apps or opening the **Settings** menu. Use this account to demo day-to-day administration without exposing the technical configuration surface. |

## Tips

- Passwords are intentionally identical to the login (e.g. `demo_student` / `demo_student`) so they're easy to remember while exploring. **Change or remove these accounts before exposing the install to anyone outside your team.**
- The student and guardian accounts are *portal* users — if you sign them in at `/web/login` you'll be redirected to the portal. They have no access to the Odoo backend.
- The teacher and registrar accounts are *internal* users and count against your Odoo user count.
- To see the underlying records, look at the **Settings → Users & Companies → Users** list while logged in as `admin`.

## Related

- [First login and initial setup →](03-first-login.md)
- [What the demo data contains →](03-first-login.md#what-the-demo-data-contains)
- [Using the /my/school dashboard →](../portal-guides/01-student-and-guardian-dashboard.md)
