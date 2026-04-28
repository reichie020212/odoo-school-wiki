---
title: odoo-school Knowledge Base
description: Top-level index for the odoo-school Wiki.js knowledge base
published: true
date: 2026-04-28
tags: index, home, overview
editor: markdown
dateCreated: 2026-04-28T05:59:30.243Z
---

# odoo-school Knowledge Base

This is the root index for the **odoo-school** Wiki.js. The content is organised into three namespaces — pick the one that matches what you need.

## Namespaces

| Namespace | What's inside | Audience |
|---|---|---|
| [**Wiki**](/wiki/home) | Getting started, admin guides, portal guides, operations, reference | End users, admins, operators |
| [**Plan**](/plan/school-base-blueprint) | Module blueprints — design decisions, scope, layout per module | Architects, reviewers |
| [**Guide**](/guide/school_base-erd) | Entity relationship diagrams (Mermaid) per module | Developers, schema reviewers |

## Quick links

- **New here?** Start with [Wiki → Getting Started](/wiki/getting-started/01-overview).
- **Looking up a schema?** Each module has an ERD under [`/guide`](/guide/school_base-erd).
- **Reviewing a design?** Each module has a blueprint under [`/plan`](/plan/school-base-blueprint).

## Modules at a glance

The system is **11 Odoo 19 Community Edition modules** covering the full student lifecycle:

`school_base` · `school_academics` · `school_exams` · `school_fees` · `school_library` · `school_canteen` · `school_shop` · `school_transport` · `school_portal` · `school_comms` · `school_homepage`

For module-by-module coverage see the [Wiki home page](/wiki/home).

## Conventions

- **Editing**: this wiki is git-backed. Edits made in Wiki.js are pushed to [`reichie020212/odoo-school-wiki`](https://github.com/reichie020212/odoo-school-wiki) on the `main` branch. Edits committed to git are pulled back into Wiki.js on the next sync tick.
- **Frontmatter is required**: every `.md` file needs a Wiki.js YAML header (`title`, `description`, `published`, `date`, `tags`, `editor`) or it will be skipped on pull.
- **Paths**: links above are Wiki.js routes (no `.md` suffix). When linking from one wiki page to another, use the same form.
