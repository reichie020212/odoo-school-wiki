# odoo-school-wiki

Wiki.js content for the [odoo-school](https://github.com/reichie020212/odoo-school) School Management System.

This repo holds a snapshot of the `docs/` tree from the application repo, kept here so Wiki.js can use it as a Git-storage backend without requiring read access to the application source.

## Layout

```
docs/
├── wiki/   — Wiki.js-ready Markdown (with YAML frontmatter + screenshots)
├── plan/   — module blueprints (dev specs)
└── guide/  — entity-relationship diagrams (ERDs)
```

`docs/wiki/` is the only tree intended for direct rendering by Wiki.js. The other two directories are reference material kept alongside the wiki for convenience.

## Wiki.js setup

Point Wiki.js's Git storage module at this repo with **Base Directory = `docs/wiki`**. See [`docs/wiki/README.md`](docs/wiki/README.md) for the full import procedure and page-to-URL mapping.

## Updating from upstream

When `reichie020212/odoo-school`'s `docs/` tree changes, mirror those changes into this repo (`docs/` here matches `docs/` there 1:1) and commit. Wiki.js picks up the changes on its next sync.
