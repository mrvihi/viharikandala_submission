# NOTES

## 1. Report — Duplicate payment webhook

**Status:** fixed

**What I saw:** Delivering the same successful payment event more than once inserted multiple payment records. A 1,200-cent payment was counted as 2,400 cents.

**Root cause:** The payments table did not enforce uniqueness for the provider event ID, and the webhook handler inserted a row for every delivery.

**Fix and why:** Added a unique index on `payments.event_id` and used `ON CONFLICT(event_id) DO NOTHING`. Repeated delivery of the same event can no longer increase the accepted payment total.

**How I verified it:** Added `test_duplicate_payment_webhook_is_not_counted_twice`. It failed before the fix and passed afterward. The final full test suite passed.

**Commits:** `1172330`

## 2. Report — Incorrect stock after abandoned checkout

**Status:** fixed

**What I saw:** Cancelling a pending order did not restore reserved stock. Separate Catalog instances could also hold stale cached stock values.

**Root cause:** Cancellation restored stock only for paid orders, not pending orders. The catalog cache could also become stale when another Catalog instance changed the database.

**Fix and why:** Updated cancellation to release stock for both pending and paid orders. Updated catalog reservation logic to refresh and invalidate its cache around stock changes.

**How I verified it:** Added tests for pending-order cancellation and stale-cache refresh. They reproduced the problems before the fix and passed afterward. The full test suite passed.

**Commits:** `0f0c0f8`

## 3. Report — Coupon tax calculation

**Status:** fixed

**What I saw:** Tax was calculated on the original subtotal instead of the discounted amount. For a 1,200-cent order with a 10% discount in CA, the old calculation produced 87 cents tax instead of the expected 78 cents.

**Root cause:** The pricing code calculated tax before subtracting the coupon discount.

**Fix and why:** Changed the calculation order to subtract the coupon discount first, then calculate tax on the discounted taxable amount.

**How I verified it:** Added `test_coupon_discount_is_applied_before_tax`. It failed before the fix and passed afterward. The full test suite passed.

**Commits:** `f4d31c9`

## 4. Report — India local-date sales reporting

**Status:** fixed

**What I saw:** A payment received at 20:00 UTC on 9 March 2025 corresponds to 01:30 on 10 March in India. The report for 10 March with a UTC offset of +330 minutes did not include that sale before the fix.

**Root cause:** The code applied the UTC offset in the wrong direction when calculating the UTC start of a shop-local day.

**Fix and why:** Changed the day-boundary calculation to subtract the shop's UTC offset from local midnight to obtain the correct UTC instant. The end boundary is one day after the start boundary.

**How I verified it:** Added `test_daily_sales_uses_india_local_date`. It failed before the fix and passed afterward.

**Commits:** `3e5924b`

## 5. Report — Concurrent stock overselling

**Status:** fixed

**What I saw:** Two workers could both read the same available stock before either updated it. Both could reserve stock based on the old value, resulting in negative inventory.

**Root cause:** Stock reservation used a read-then-write sequence. The stock check and decrement were not one atomic database operation.

**Fix and why:** Replaced the separate check-and-decrement operation with a conditional SQL update that subtracts stock only when enough stock is available. The affected-row count determines whether the reservation succeeded.

**How I verified it:** Added `tests/test_catalog_concurrency.py`. It synchronizes two workers attempting to reserve 4 units when only 5 are available. The expected result is one successful reservation, one `OutOfStock` error, and 1 unit remaining. The concurrency test and full test suite passed.

**Commits:** `6a21969`

## 6. Open question — Two separate successful payments

**My decision:** First-payment-wins. Accept payments only up to the remaining order balance. Flag extra successful payments for refund.

**Why:** A repeated delivery of the same event is different from a customer making a second, separate payment. The first problem is handled through event-ID idempotency. A genuinely separate extra payment should be recorded but should not make the order appear to have received more money than its total.

**What it costs:** The application flags the payment but does not automatically issue a refund through the payment provider. Someone must review and process the refund manually. If an event exceeds the remaining balance, the entire event is flagged rather than splitting it into an accepted portion and a refund portion.

**What I built:** Added a `refund_required` column to the payments table, defaulting to 0. Accepted payments count toward the order balance. Payments received after the order is paid, or payments exceeding the remaining balance, are recorded with `refund_required = 1` and excluded from `paid_cents()`. Partial payments can combine to complete an order. Payment decisions are serialized with `BEGIN IMMEDIATE` so concurrent webhook handlers cannot both accept payments against the same remaining balance. Existing databases are upgraded with the new column, and existing payment records default to 0.

**How I verified it:** Added tests for a second distinct successful payment, partial payments combining to pay an order, and an overpayment being flagged. The second-payment test failed before the fix because the order showed 2,400 cents received for a 1,200-cent order. All three tests passed afterward. Also tested opening an old database and confirmed that the migration adds `refund_required` with a default of 0. The final full test suite passed all 16 tests.

**Commits:** `9ca5937`

## Other things I noticed

The test suite initially showed SQLite connection ResourceWarnings because test connections were not being closed. I added test teardown cleanup to close the connection in `tests/test_orders.py`. No other unrelated application issues were investigated or changed.

## Session logs

AI tool used: ChatGPT.

Full conversation exports must be included in `session_logs/`, one file per assessment conversation. The assessment work was discussed in conversations titled `Engineering Assessment Plan` and `Engineering Assessment Plan (1)` through `Engineering Assessment Plan (7)`. The files must contain the full conversations, not summaries.
