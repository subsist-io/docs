# Transactions

A **transaction** is one billing event for one subscription: a trial starting, a first payment, a renewal, a refund. It is Subsist's normalized record, derived from a store [receipt](receipts.md). Whatever store the money came through, a transaction has the same shape, which is what lets finance, growth and support work from one table.

Transactions are derived data. If a store updates a receipt, the transaction rows for it are updated to match. Nobody edits them by hand.

## The `Transaction` table

Defined in `payments/models.py`.

| Field | Type | Meaning |
|---|---|---|
| `merchant` | text | Store that billed it: `Apple`, `Google`, `Amazon`, `Roku` |
| `transaction_id` | text | The store's ID for this billing event. Together with `merchant`, this is the row's identity |
| `subscription_id` | text | The store's ID for the subscription this event belongs to. Every renewal of one subscription shares it |
| `sku` | text | The store product ID the customer bought |
| `brand` | text | Which of the customer's apps this came from (for example the Apple bundle ID or Google package name) |
| `external_user_id` | text | The customer app's own user ID, copied from the receipt |
| `payment_date` | datetime (UTC) | When the store charged, or when the trial started |
| `payment_amount` | decimal(20, 4) | Amount charged, in `currency_code` |
| `currency_code` | text | ISO 4217 currency code |
| `payment_status` | text | See [Payment status](#payment-status) |
| `renewal_status` | text | See [Renewal status](#renewal-status) |
| `in_free_trial` | boolean | True if this period is a free trial |
| `billing_cycle_start` | datetime (UTC) | Start of the service period this event pays for |
| `billing_cycle_end` | datetime (UTC) | End of that period; for a refund, when access was revoked |
| `create_dt` | datetime | When Subsist first recorded the row |

There is a composite index on (`transaction_id`, `create_dt`).

## Payment status

What happened to the money for this period.

| Value | Meaning |
|---|---|
| `Free Trial` | Service period with no charge because of a free-trial offer |
| `Intro Offer` | Discounted introductory period (pay-as-you-go or pay-up-front) |
| `Paid` | The store collected payment for the period |
| `Refunded` | The store reversed the payment; `billing_cycle_end` is the revocation time |

A subscriber in billing retry, grace period or account hold has no new paid transaction for the period; that state lives on the subscription's [lifecycle](lifecycle.md), not in this table.

## Renewal status

Whether the subscription is set to renew after `billing_cycle_end`, as of the last sync.

| Value | Meaning |
|---|---|
| `Active` | Auto-renew is on |
| `Canceled` | The customer turned auto-renew off. They keep access until `billing_cycle_end`, then churn |

Renewal status is a property of the subscription, not the individual payment, so it is rewritten on every row of a subscription at each sync.

## How each store maps into a transaction

| Field | Apple | Google Play | Amazon | Roku |
|---|---|---|---|---|
| `transaction_id` | `transactionId` | `lineItems.latestSuccessfulOrderId` | `receiptId` | `transactionId` |
| `subscription_id` | `originalTransactionId` | The `purchaseToken` chain (a token plus its `linkedPurchaseToken` ancestors). Don't parse the order ID; Google doesn't document its format | `receiptId` | `originalTransactionId` |
| `sku` | `productId` | `lineItems.productId` | `termSku` | `productId` (the purchase-option SKU) |
| `brand` | `bundleId` | `packageName` | `appPackageName` (from Real-time Notifications) | `channelId` |
| `payment_amount` / `currency_code` | `price` ÷ 1000 / `currency` (informational only per Apple) | `orders.get` → `total` (after discounts and tax) | Not returned by RVS | `total` / `currency` |
| `in_free_trial` | `offerDiscountType` = `FREE_TRIAL` | `offerPhase.freeTrial` | `freeTrialEndDate` is non-null | `isFreeTrial` (push notification) |
| `renewal_status` | Renewal info `autoRenewStatus` (1 = on) | `autoRenewingPlan.autoRenewEnabled` | `autoRenewing` | `cancelled` = false |
| `billing_cycle_end` | `expiresDate` | `lineItems.expiryTime` | `renewalDate` | `expirationDate` |
| Refund signal | `revocationDate` / `revocationReason` | Voided Purchases API | `cancelReason` / RVS 410 | Refund push notification / `validate-refund` |

Source details, field names and citations are on each store page: [Apple](../sources/apple.md), [Google Play](../sources/google.md), [Amazon](../sources/amazon.md), [Roku](../sources/roku.md).

## Rules

- **One row per billing event.** A monthly subscription that renews twelve times has twelve rows with the same `subscription_id`.
- **Upsert, never duplicate.** Rows are looked up by (`merchant`, `transaction_id`) and updated in place when the store's view changes.
- **All times in UTC.** Store timestamps (milliseconds since epoch for Apple, Google and Amazon; ISO strings for Roku) are converted to aware UTC datetimes.
- **Unknown is null, not a sentinel.** If a store does not report a value (Amazon never returns price), leave it empty rather than writing `"n/a"` or `0`.
- **Sandbox is separate.** Test and sandbox purchases are recorded with their environment so they never mix into production revenue.

## Known gaps in the current code

- `payment_amount` and `currency_code` are nullable (migration `payments/0003`), because Amazon's RVS returns no price.
- `Meta.index_together` was replaced with a named `Meta.indexes` entry (migration `payments/0004`) as part of the move to Django 5.2.
- There is no unique constraint on (`merchant`, `transaction_id`). Add one so the upsert rule is enforced by the database.
- `payment_status` and `renewal_status` are free text. Turning them into `TextChoices` would stop adapters from inventing new values.
- Amazon and Roku adapters write placeholder values (`"unknown"`, `"brand"`) into typed columns. See each store page's migration notes.
