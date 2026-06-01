# Settings Panel Reference (Sprint 9)

Living reference for the admin Settings panel at `/admin/settings`. Each
section lists: what the setting does, who sets it, where it takes effect,
and a recommended starting value.

All settings below are Owner-only unless noted. Changes apply to *new*
checkouts / orders immediately — historical orders keep their snapshotted
values (shipping, discount, COD fee are all frozen at placement time).

---

## 1. Store & invoice info — `/admin/settings/store`

| Field | Appears on | Default |
|---|---|---|
| Company name (AR/EN) | Invoice header, emails, storefront footer | Print By Falcon |
| Commercial registry # | Invoice header | — (owner-supplied) |
| Tax card # | Invoice header | — (owner-supplied) |
| Company address (AR/EN) | Invoice header | Store headquarters |
| Phone | Invoice + footer | Store business number |
| Email | Invoice + footer | admin@printbyfalcon.com |
| Website | Invoice footer | printbyfalcon.com |

**Note.** Prior invoices keep the values that were current when they were
generated. Editing this page does NOT rewrite historical PDFs. Amendments
(Sprint 6) produce a new version carrying the updated header + the "Amended"
watermark.

## 2. Shipping & zones — `/admin/settings/shipping`

Five admin-editable zones. Each zone carries:

| Field | Meaning |
|---|---|
| Shipping rate (EGP) | Flat fee added at checkout when subtotal < free-shipping threshold |
| Free-shipping threshold — B2C | `subtotal ≥ threshold ⇒ shipping = 0` for non-B2B viewers. Empty = no free shipping |
| Free-shipping threshold — B2B | Same as above, applied when viewer is an active B2B customer |
| COD available | Per-zone toggle — when off, COD radio is hidden even if the global policy is enabled |

**Defaults** (Sprint 9 kickoff 2026-04-22):

| Zone | Rate | Free B2C | Free B2B | COD |
|---|---|---|---|---|
| Greater Cairo (CAIRO, GIZA, QALYUBIA) | 50 | 1,500 | 5,000 | ✅ |
| Alex + Delta (9 govs) | 70 | 1,500 | 5,000 | ✅ |
| Canal + Suez (ISMAILIA, PORT_SAID, SUEZ) | 70 | 1,500 | 5,000 | ✅ |
| Upper Egypt (8 govs) | 90 | 1,500 | 5,000 | ✅ |
| Sinai + Red Sea + Remote (5 govs) | 120 | 1,500 | 5,000 | ✅ |

**Bulk mapping UX.** Tick several governorates in the lower table, pick a
target zone in the dropdown, click Apply — one round-trip updates them all.
PRD Q#10 closed at Sprint 9 kickoff with the table above.

## 3. Cash-on-delivery policy — `/admin/settings/cod`

Global COD policy, layered on top of the per-zone toggle in §2.

| Field | Meaning |
|---|---|
| Enabled | Master switch — off hides COD from all zones |
| Fee kind | `FIXED` (EGP amount) or `PERCENT` (of subtotal − discount + shipping) |
| Fee value | Amount or percentage value |
| Max order (EGP) | Orders above this pre-fee total are rejected for COD |

**Defaults:** enabled, fee = 20 EGP fixed, max order = 15,000 EGP.

The COD fee is itself captured on `Order.codFeeEgp` so the COD reconciliation
report (§8) can split "goods collected" vs. "COD fees collected" when ops
meets the courier.

## 4. Promo codes — `/admin/promo-codes`

List + create + edit. Fields per code:

| Field | Meaning |
|---|---|
| Code | Case-insensitive at input, stored uppercase. Must be unique |
| Type | `PERCENT` (0–100) or `FIXED` (EGP) |
| Value | The discount amount |
| Min order (EGP) | Optional — promo rejected when subtotal < min |
| Usage limit | Optional — promo deactivates after N redemptions |
| Valid from / until | Optional — inclusive window |
| Active | Master on/off |
| Description | Internal note, not shown to customer |

**Stacking rule (MVP):** one promo code per order. Atomic `usedCount`
increment happens inside the order-placement transaction — a racing second
checkout for the last redemption sees the DB already at the cap and fails
with `promo.usage_limit_reached`.

**Bulk "Disable expired"** on the list page deactivates every active code
whose `validTo` is in the past — keeps the list clean without clicking
through.

**Usage stats** on the edit page show: total redemptions, number of orders,
total discount granted. Updated on every order placement.

Seed 3 demo codes with `npm run seed:promo` (WELCOME10 / SAVE50 / RAMADAN26
— the last is pre-expired for UI demo).

## 5. VAT — `/admin/settings/vat`

Default 14% (Egypt standard). Per-product `vatExempt` override lives on
the product edit page.

**MVP note.** Sprint 9 surfaces the setting + per-product exempt override
but does NOT retrofit VAT into checkout totals — existing `Order.vatEgp`
stays at 0. Breaking VAT out of the checkout total and the invoice line
is on the v1.1 roadmap per PRD.

## 6. Inventory thresholds — `/admin/settings/inventory`

Global low-stock threshold (default 5 per ADR-035). Per-SKU override on
the product edit page. Both feed the admin low-stock widget + the daily
digest.

## 7. Notifications — `/admin/settings/notifications`

Per-status × channel opt-out matrix (Sprint 5). Turning a cell off stops
the notification from firing for that status + channel combination across
every future order. Historical orders' queued notifications are unaffected.

## 8. Courier partners — `/admin/couriers`

CRUD for the courier list that surfaces in the "Hand to courier" modal on
`/admin/orders/[id]`. AR + EN names, default contact phone, active toggle,
sort position. Deactivated couriers disappear from the dropdown but stay
linked to historical orders for reporting.

## 9. COD reconciliation report — `/admin/orders/cod-reconciliation`

Ops workflow. Lists every order in `payment_method = COD` +
`payment_status = PENDING_ON_DELIVERY`, grouped by courier partner, with
totals for goods, COD fees, and total cash to collect. Date-range
filterable (defaults to the last 30 days).

When the courier hands the cash back, ops opens the order detail
(`/admin/orders/[id]`) and clicks **Mark cash received** (green button in
the Payment section) — that flips the order's payment status to `PAID` and
logs the note + actor in the audit trail.
