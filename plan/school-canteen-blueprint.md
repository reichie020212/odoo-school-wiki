---
title: school_canteen — Module Blueprint
description: Module blueprint for school_canteen
published: true
date: 2026-04-28
tags: blueprint, plan, school_canteen
editor: markdown
---

# `school_canteen` — Module Blueprint

**Status:** autonomous execution under Red's blanket approval 2026-04-24.
**Depends on:** `school_base`, `product`, `mail`.

## Decisions

1. **Canteen items are products.** `product.template` extended with `is_canteen_item` + `canteen_dietary_tag_ids`. Reuses Odoo pricing, taxonomy, and images.
2. **Dietary tags are shared** between "item contains X" and "student is allergic to X" — `school.canteen.dietary.tag` attaches to both. Purchase blocks when the two sets intersect.
3. **Wallet per student, single-record.** Unique (student) constraint. Balance is stored (updated on transaction post).
4. **Transactions** are a single model with a type (`topup` or `purchase`), draft → posted → cancelled.
5. **Menus** are daily rosters: `school.canteen.menu` has `date`, state, and a m2m of products. Optional in the flow — a purchase doesn't have to be from today's menu.

## Models

| Model | Purpose |
|---|---|
| `school.canteen.dietary.tag` | Tag (Nuts, Gluten, Dairy…). Shared between items and student allergies. |
| `product.template` (inherit) | `is_canteen_item` bool + `canteen_dietary_tag_ids` m2m. |
| `res.partner` (inherit) | `canteen_allergy_tag_ids` m2m (surface only in Student Info on is_student). |
| `school.canteen.menu` | Daily menu: date, name, items, state. |
| `school.canteen.wallet` | student_id unique, balance, topup_total, purchase_total, currency, status (`active`/`frozen`). |
| `school.canteen.transaction` | wallet_id, type (topup/purchase), amount, date, item_ids (for purchase), description, state, posted_at. |

## Flow

1. Parent tops up student's wallet: create `school.canteen.transaction` type=topup, amount=X → post → wallet.balance += X.
2. Student buys: new transaction type=purchase with items, amount auto-computed (sum of product list prices) → post.
   - Allergy constraint: student's allergy tags ∩ items' dietary tags must be empty.
   - Balance constraint: wallet.balance ≥ amount.
   - On post: wallet.balance -= amount.
3. Cancel a posted transaction reverses the balance change.

## Menus

- **School → Canteen → Wallets / Transactions / Daily Menus**
- **School → Configuration → Dietary Tags / Canteen Items** (manager)

## Tests

- Wallet uniqueness per student.
- Topup increases balance on post.
- Purchase decreases balance on post.
- Insufficient balance rejected.
- Allergy intersection blocks the purchase.
- Cancel reverses balance.
- Dietary tag code uniqueness.
