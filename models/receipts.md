# Receipts

A **receipt** is whatever a store gives us to prove a purchase happened. Subsist stores the receipt exactly as the store returned it, then derives [transactions](transactions.md) and [lifecycle](lifecycle.md) events from it. The receipt is the raw evidence; transactions are our normalized interpretation of it.

Receipts are never edited by hand. When a store changes its mind (a refund, a renewal, a cancellation), we re-fetch the receipt and store the new response. Transactions are then re-derived.

## Why receipts are hard

Each store has a different idea of what a "receipt" is:

| Store | What the app sends us | What identifies the subscription | What identifies the person |
|---|---|---|---|
| [Apple](../sources/apple.md) | A signed transaction (JWS) or, on the legacy path, a base64 app receipt | `originalTransactionId` | `appAccountToken` (UUID the app sets at purchase) |
| [Google Play](../sources/google.md) | A `purchaseToken` plus package name | The chain of `purchaseToken`s joined by `linkedPurchaseToken` | `obfuscatedExternalAccountId` (set by the app at purchase) |
| [Amazon](../sources/amazon.md) | A `receiptId` plus the Amazon `userId` | `receiptId` (stays the same across renewals of a continuous subscription) | Amazon `userId` |
| [Roku](../sources/roku.md) | A `transactionId` | `originalTransactionId` | `rokuCustomerId`, or `partnerReferenceId` if the app set it |

Things that follow from this, all of which Subsist has to handle:

- One person can have several receipts (different devices, different stores, re-subscribing after a lapse).
- One receipt can be presented by several of the app's user accounts (shared devices, account switching, family sharing).
- We can have a receipt and not know the person, or know the person and not yet have the receipt.
- A receipt's identifier can change for the same subscription. Google issues a new `purchaseToken` on upgrade, downgrade or re-signup; Apple issues a new `transactionId` every renewal.

## Fields common to every receipt

These live on `BaseReceipt` in `payments/models.py` and every store's receipt table inherits them.

| Field | Type | Meaning |
|---|---|---|
| `merchant` | text | Which store issued it: `apple`, `apple-sandbox`, `google`, `amazon`, `roku` |
| `external_user_id` | text, nullable, indexed | The customer app's own user ID, if we know it |
| `create_dt` | datetime | When Subsist first saw the receipt |
| `mod_dt` | datetime | When the row last changed |
| `sync_dt` | datetime, nullable | When the receipt was last re-checked with the store |

Every store receipt also has a `merchant_response` JSON column holding the **complete, unmodified** store response from the last sync. Keep it whole: store APIs add fields over time, and a full copy lets us re-derive transactions later without calling the store again.

## Per-store receipt tables

| Store | Table | Key fields beyond the common ones |
|---|---|---|
| Apple | `apple.AppleReceipt` | One row per subscription: `original_transaction_id`, `app` (credentials in `apple.AppleApp`), `environment`, `status` (1–5), `auto_renew_status`, `expires_dt`, `app_account_token`; `receipt` holds a legacy base64 app receipt until it is converted. Notifications are logged in `apple.AppleNotification` |
| Google | `google.GoogleReceipt` (package `google_play`) | `purchase_token` (primary key), `app` (credentials in `GooglePlayApp`), `subscription_state`, `auto_renew_enabled`, `expires_dt`, `base_plan_id`, `offer_id`, `obfuscated_external_account_id`, `linked_purchase_token` / `subscription_root_token` / `superseded_by` (the token chain). Notifications are logged in `GoogleNotification` |
| Amazon | `amazon.AmazonReceipt` | `merchant_user_id`, `receipt_id` (unique), `app` (credentials in `AmazonApp`), the documented RVS subscription fields (`auto_renewing`, renewal/cancel/free-trial/grace-period dates, `cancel_reason`), `status`, `environment`, and `subscription_root_id` for plan-change chains. Notifications are logged in `AmazonNotification` |
| Roku | `roku.RokuReceipt` | Flattened `validate-transaction` fields: `transaction_id`, `original_transaction_id`, `product_id`, `purchase_status`, `is_entitled`, `cancelled`, `expiration_date`, `amount`, `tax`, `total`, `currency`, `roku_customer_id`, `partner_reference_id`, and error fields |

The Roku table flattens the response into columns; every store also keeps the full response in `merchant_response`.

## How a receipt moves through Subsist

1. **Intake.** The customer's app (or backend) sends Subsist the store identifier from the table above, along with its own `external_user_id` if it has one.
2. **Verify.** Subsist calls the store to validate it and saves `merchant_response`. A receipt that fails validation is kept, flagged, and not turned into transactions.
3. **Derive.** Each store's adapter reads `merchant_response` and upserts rows in the [transactions](transactions.md) table.
4. **Re-sync.** Receipts are re-checked on a schedule (`sync_dt`), and immediately whenever the store sends a server notification. Each store page describes its re-sync endpoint and cadence:
   - Apple: Get All Subscription Statuses and Get Transaction History ([Apple](../sources/apple.md))
   - Google: `purchases.subscriptionsv2.get` ([Google Play](../sources/google.md))
   - Amazon: Receipt Verification Service ([Amazon](../sources/amazon.md))
   - Roku: `validate-transaction`, nightly for subscriptions at or past expiry ([Roku](../sources/roku.md))

## Current state of the code (2022) vs. the store APIs (2026)

Every store adapter still targets the API that existed in 2022. The store pages list the details; in short:

| Store | Code calls | Current API |
|---|---|---|
| Apple | App Store Server API + Notifications V2 (migrated 2026-09-28; `verifyReceipt` no longer called) | Same |
| Google | v3 `purchases.subscriptionsv2.get` + RTDN (migrated 2026-09-28) | Same |
| Amazon | RVS 1.0 + Real-time Notifications via SNS (migrated 2026-09-28) | Same |
| Roku | Roku Pay web services + signed push notifications + Enhanced Subscription Recovery (migrated 2026-09-28) | Same |

All four stores' server notifications are now received and verified. Adding a notification intake per store is the biggest gap between the receipt model and how the stores expect to be integrated today.
