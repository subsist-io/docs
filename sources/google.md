# Google Play

> Last reviewed: 2026-09-28 · Current versions: Play Billing Library 9.1.0 (released 2026-06-18) [1]; Google Play Developer API v3 (latest release-notes entry 2026-07-06, discovery revision 20260928) [4][23]

## Overview

Google Play organizes subscriptions in three layers. A **subscription** is the product (the set of benefits a user is entitled to), identified by a product ID. Each subscription has one or more **base plans**, and each base plan defines a billing period, a renewal type (auto-renewing, prepaid, or installments) and a price per region. Each auto-renewing base plan can have one or more **offers**, which give eligible users a discount through one or more offer phases, such as a free trial followed by an introductory price. Offers can only be created for auto-renewing base plans, and a subscription can have up to 250 base plans and offers combined, with at most 50 active at once [20]. Base plan IDs cannot be changed or reused once activated [21].

A purchase is identified by a **purchase token**. The server-side source of truth for a token is `purchases.subscriptionsv2.get`, which returns a `SubscriptionPurchaseV2` resource with a top-level `subscriptionState` and one or more `lineItems` (one per product; more than one when the purchase is a subscription with add-ons) [6][11]. Each renewal produces a new order ID but keeps the same purchase token; upgrades, downgrades, re-signups and prepaid top-ups produce a new purchase token that points back to the old one through `linkedPurchaseToken` [6][9][11]. Google pushes state-change signals through Real-time developer notifications (RTDN) on Cloud Pub/Sub. These notifications only say that something changed, so the backend must always call the Developer API to get the full state [9][10].

## APIs and versions

| API / library | Current version | What Subsist uses it for | Status |
|---|---|---|---|
| Google Play Billing Library (Android client) | 9.1.0, released 2026-06-18. 9.0.0 released 2026-05-19; the migration guide from 7/8 to 9 is at [3] [1] | Not called by Subsist. Relevant because the customer's app must call `setObfuscatedAccountId` so that Subsist can link purchases to users, and because client-side behavior (for example, suspended subscriptions from 8.1.0) affects what users see [1][11] | Current. Every version has a two-year deprecation cycle. New apps and updates must use: v7 or later by 2025-08-31 (v6 extension to 2025-11-01); **v8 or later by 2026-08-31** (v7 extension to **2026-11-01**); v9 or later by 2027-08-31 (extension to 2027-11-01); v10 or later by 2028-08-31 for v9 (extension to 2028-11-01) [2]. As of today, v7 is only allowed for apps with an approved extension. |
| Google Play Developer API (`androidpublisher`) | v3 [8][23] | All server-side calls | Current. v1 and v2 were shut down on 2019-12-01 [22]. |
| `purchases.subscriptionsv2` (`get`, `cancel`, `defer`, `revoke`) | v3 | Main status read (`get`), re-sync, and developer-initiated actions | Current. The documentation calls `get` "the source of truth" [11]. `cancel` was added 2025-09-11 [4]. |
| `purchases.subscriptions` (v1 resource) | v3 | `acknowledge` only | **Partly deprecated.** `get`, `refund` and `revoke` were deprecated on 2025-05-21 and shut down on **2027-08-31** (extension available until 2027-11-01). Client libraries released after 2026-07-01 no longer include them [5]. `cancel` and `defer` were deprecated on 2026-05-19 and shut down on 2028-08-31 (extension until 2028-11-01) [5]. `acknowledge` is **not** deprecated and is still the way to acknowledge from the server [7][8][11]. The v3 reference [8] and the live discovery document (revision 20260928) [23] now list only `acknowledge`, `cancel` and `defer` on this resource. |
| `purchases.voidedpurchases.list` | v3 | Backfill and reconciliation of refunds, chargebacks and revocations | Current [15][16] |
| `orders` (`get`, `batchget`, `refund`, `reviewrefund`) | v3 | Transaction amounts (total, tax, developer revenue), order state, refunds. Replaces `purchases.subscriptions.refund` [5] | Current [17]. `get` and `batchGet` were added 2025-05-21; `reviewrefund` was added 2026-07-06 [4]. The field `Order.lineItems.subscriptionDetails.offer_phase` is deprecated (2026-05-19, shut down 2028-08-31) in favor of `offer_phase_details` [5]. |
| `monetization.subscriptions`, `.basePlans`, `.basePlans.offers` | v3 | Reading the catalog (products, base plans, offers, billing periods, grace/hold settings) to label ledger rows | Current. `monetization.subscriptions.archive` is deprecated ("subscription archiving is not supported") [8]. |
| Real-time developer notifications (Cloud Pub/Sub) | Notification `version` "1.0" [9] | Event trigger for every lifecycle change | Current. `SUBSCRIPTION_PRICE_CHANGE_CONFIRMED` (8) and the `SubscriptionNotification.subscriptionId` field are deprecated (2025-05-21, shut down 2027-08-31) [5]. |

## Credentials required

A Subsist customer must provide or set up the following.

1. **Package name** of each Android app (for example `com.some.thing`). Every Developer API call is scoped to `applications/{packageName}` [8].
2. **A Google Cloud project with the Google Play Developer API enabled.** This is done in Google Cloud Console. Linking the Play developer account to a Cloud project is no longer required [18].
3. **A service account JSON key.** Create the service account in Google Cloud Console (IAM > Service Accounts), then create and download a JSON key for it [18]. The OAuth scope is `https://www.googleapis.com/auth/androidpublisher` [16].
4. **Play Console access for that service account.** In Play Console, go to *Users & Permissions*, choose *Invite new users*, enter the service account's email, and grant at least **View financial data, orders, and cancellation survey responses** and **Manage orders and subscriptions** [18]. The Voided Purchases API specifically needs permission to view financial data [15]. Without these permissions, calls fail with `403 userInsufficientPermission` [6].
5. **RTDN Pub/Sub topic** (see [Server notifications](#server-notifications)). The customer creates the topic in their Cloud project, grants Google Play's publisher account access, and enters the full topic name in Play Console under *Monetize > Monetization setup* [10]. Subsist then needs either a push subscription pointed at its endpoint, or a pull subscription plus IAM credentials that can pull from it.

**Old approach versus service accounts.** The 2022 Subsist code used an OAuth 2.0 web-server client: a `client_id`, a `client_secret` and a long-lived `refresh_token`, exchanged at `https://accounts.google.com/o/oauth2/token` for short-lived access tokens [19]. Google still documents that flow [19], but the current getting-started guide says that "in most cases" you should use a service account, and describes OAuth clients as the option for acting "on behalf of an individual user" [18]. A refresh token is tied to one human Google account and stops working if that person leaves or loses access. A service account is a non-human identity with permissions granted in Play Console. Subsist should accept a service account JSON file and treat the OAuth refresh-token path as legacy.

## Server notifications

### Setup

1. **Create a topic** in the customer's Cloud project (the Cloud Pub/Sub API must be enabled) [10].
2. **Create a subscription** on the topic. With **push**, Pub/Sub sends HTTPS requests to Subsist's endpoint, and a successful response code acknowledges the message. With **pull**, Subsist calls Pub/Sub to fetch messages and must acknowledge each one to avoid redelivery. Google suggests push if you are unsure [10].
3. **Grant publish rights.** On the topic's permissions, add `google-play-developer-notifications@system.gserviceaccount.com` with the role **Pub/Sub Publisher**. If domain-restricted sharing is enforced in the organization, an exception is needed for this account [10].
4. **Enable RTDN in Play Console.** Go to *Monetize > Monetization setup > Real-time developer notifications*, tick *Enable real-time notifications*, and enter the topic as `projects/{project_id}/topics/{topic_name}`. Use *Send Test Message* to verify the setup. Then choose either "subscriptions and all voided purchases" or "all notifications for subscriptions and one-time products" [10].
5. RTDN is configured **per app**. Different apps may use different Cloud projects for their topics, but all apps in a developer account must use the same Cloud project for the Developer API [10].

Each notification is about 1 KB of data [10]. Pub/Sub's `messageId` is unique per notification, and Google recommends de-duplicating on it [9].

### Message format

The Pub/Sub envelope carries a base64-encoded `data` field [9]:

```json
{
  "message": {
    "attributes": { "key": "value" },
    "data": "eyAidmVyc2lvbiI6IHN0cmluZywg...",
    "messageId": "136969346945"
  },
  "subscription": "projects/myproject/subscriptions/mysubscription"
}
```

The decoded `DeveloperNotification` has `version` (currently `"1.0"`), `packageName`, `eventTimeMillis`, and exactly one of `subscriptionNotification`, `oneTimeProductNotification`, `voidedPurchaseNotification`, `pendingRefundReviewNotification` or `testNotification` [9]. A `SubscriptionNotification` contains only `version`, `notificationType` (int) and `purchaseToken` [9]:

```json
{
  "version": "1.0",
  "packageName": "com.some.thing",
  "eventTimeMillis": "1503349566168",
  "subscriptionNotification": {
    "version": "1.0",
    "notificationType": 4,
    "purchaseToken": "PURCHASE_TOKEN"
  }
}
```

The older `subscriptionId` field of `SubscriptionNotification` is deprecated with no replacement [5]. Get the product from `lineItems[].productId` in the `subscriptionsv2.get` response instead.

### SubscriptionNotification types

Names and codes are copied from the RTDN reference [9]. The codes are not contiguous: 14, 15, 16 and 21 are not listed. Subsist should log unknown codes and still re-sync the token. "Subsist event" is the recommended mapping, and it should always be confirmed against the `subscriptionsv2.get` response.

| Code | Name | Meaning (per Google) | Subsist event |
|---|---|---|---|
| 1 | `SUBSCRIPTION_RECOVERED` | Recovered from account hold, or resumed from pause [9]. Sent after a pause ends (automatically or manually) and after a payment fix during account hold [11]. | `subscription.resumed`. Also emit `payment.renewed` if `lineItems[].latestSuccessfulOrderId` changed. |
| 2 | `SUBSCRIPTION_RENEWED` | An active subscription was renewed. For installment plans, sent on each billing-date charge [9][11]. | `payment.renewed`. Also emit `trial.converted` if the previous offer phase was `freeTrial`. |
| 3 | `SUBSCRIPTION_CANCELED` | Voluntary or involuntary cancellation. For voluntary cancellation, sent when the user cancels [9]. Also sent when account hold ends without recovery, and when an opt-in price increase is not accepted [11]. | `renewal.disabled`. If `expiryTime` is already in the past (for example, cancelled during hold), a 13 follows [11]. |
| 4 | `SUBSCRIPTION_PURCHASED` | A new subscription was purchased [9]. Also sent for every prepaid top-up, for upgrades/downgrades, for resubscribe after expiry, and when a pending purchase completes [11][12]. | `subscription.started`, or `trial.started` if `offerPhase.freeTrial` is set. Add `offer.redeemed` if `offerDetails.offerId` or `signupPromotion` is present. If `linkedPurchaseToken` is set, add `plan.changed` (upgrade/downgrade) or `renewal.enabled` (re-signup before expiry). |
| 5 | `SUBSCRIPTION_ON_HOLD` | Entered account hold (if enabled) [9]. Also sent when resuming from pause fails to charge [11]. | `subscription.on_hold` |
| 6 | `SUBSCRIPTION_IN_GRACE_PERIOD` | Entered grace period (if enabled) [9] | `grace_period.started`. Also emit `payment.retrying`, because Google retries the charge during grace [11]. |
| 7 | `SUBSCRIPTION_RESTARTED` | User restored a cancelled but not yet expired subscription from Play > Account > Subscriptions [9]. Same purchase token; cancellation fields are cleared [11]. | `renewal.enabled` |
| 8 | `SUBSCRIPTION_PRICE_CHANGE_CONFIRMED` (DEPRECATED) | User confirmed a price change [9]. Deprecated in favor of 19, and still sent for subscriptions without add-ons [11]. | `price.changed`. De-duplicate against 19. |
| 9 | `SUBSCRIPTION_DEFERRED` | Recurrence time was extended [9] (developer called `defer`) [11] | — (update `expiryTime` only) |
| 10 | `SUBSCRIPTION_PAUSED` | Subscription has been paused (the pause is now in effect) [9][11] | `subscription.paused` |
| 11 | `SUBSCRIPTION_PAUSE_SCHEDULE_CHANGED` | Pause schedule changed. Sent when the user initiates a pause; the subscription stays `ACTIVE` until the next renewal date [9][11]. | — (optionally record as a scheduled pause) |
| 12 | `SUBSCRIPTION_REVOKED` | Revoked before expiration time [9]. For example via `subscriptionsv2.revoke` or a chargeback. State becomes `SUBSCRIPTION_STATE_EXPIRED` [11]. | `revoked`. Add `refund.issued` when a matching voided purchase or refund appears. |
| 13 | `SUBSCRIPTION_EXPIRED` | Subscription has expired [9] | `subscription.expired` |
| 17 | `SUBSCRIPTION_ITEMS_CHANGED` | An item in a subscription bundle has been changed [9] | `plan.changed` |
| 18 | `SUBSCRIPTION_CANCELLATION_SCHEDULED` | Installment subscription cancellation scheduled for the end of the commitment period [9]. State stays `ACTIVE`, with an empty `pendingCancellation` object [11]. | `renewal.disabled` |
| 19 | `SUBSCRIPTION_PRICE_CHANGE_UPDATED` | A subscription item's price change details were updated [9]. Sent when a price change is added and on any status change, including user acceptance [11]. | `price.changed` |
| 20 | `SUBSCRIPTION_PENDING_PURCHASE_CANCELED` | A pending transaction was cancelled [9]. The user never gained access (initial purchase) or keeps the old subscription (top-up or plan change) [12]. | — |
| 22 | `SUBSCRIPTION_PRICE_STEP_UP_CONSENT_UPDATED` | Consent period for a price step-up began, or the user consented. Sent only in regions where step-up consent is required [9]. Currently this means South Korea [11]. | — (a price step-up is an offer-phase transition, not a developer price change [11]) |

Subsist events with no direct RTDN: `payment.retrying` has no dedicated notification and is inferred from 6, and from 5 because Google keeps retrying during account hold [11]. `refund.issued` comes from `VoidedPurchaseNotification` or the Voided Purchases API, not from `SubscriptionNotification`.

### Other notification types

- **`OneTimeProductNotification`**: `ONE_TIME_PRODUCT_PURCHASED` (1) and `ONE_TIME_PRODUCT_CANCELED` (2). Only sent if the app opted into one-time product events. Contains `purchaseToken` and `sku` [9][10].
- **`VoidedPurchaseNotification`**: contains `purchaseToken`, `orderId`, `productType` (`PRODUCT_TYPE_SUBSCRIPTION` = 1, `PRODUCT_TYPE_ONE_TIME` = 2) and `refundType` (`REFUND_TYPE_FULL_REFUND` = 1, `REFUND_TYPE_QUANTITY_BASED_PARTIAL_REFUND` = 2) [9]. For subscriptions, `orderId` identifies which renewal was voided. Maps to `refund.issued`, and to `revoked` if the entitlement is removed.
- **`PendingRefundReviewNotification`** (added 2026-07-06): a chargeback that needs the developer's input. Contains `pendingRefundToken`, `orderId` and `refundReason` (only `CHARGEBACK` = 7 today). Respond within 24 hours via `orders.reviewrefund` [4][9]. No Subsist lifecycle event; record it for later matching with the voided purchase.
- **`TestNotification`**: only `version`. Sent from Play Console's *Send Test Message* [9][10]. Acknowledge the message and ignore it.

## Lifecycle rules and edge cases

### `subscriptionState` values (subscriptionsv2)

Quoted from the resource reference [6]:

| Value | Meaning |
|---|---|
| `SUBSCRIPTION_STATE_UNSPECIFIED` | Unspecified. |
| `SUBSCRIPTION_STATE_PENDING` | Created but awaiting payment during signup. `startTime` is not set. |
| `SUBSCRIPTION_STATE_ACTIVE` | Auto-renewing: at least one item has `autoRenewEnabled` and is not expired. Prepaid: at least one item is not expired. |
| `SUBSCRIPTION_STATE_PAUSED` | Paused (auto-renewing only). `pausedStateContext.autoResumeTime` gives the resume time. |
| `SUBSCRIPTION_STATE_IN_GRACE_PERIOD` | In grace period (auto-renewing only). User keeps access. `inGracePeriodStateContext.renewalDeclined.pendingOrderId` gives the failed order. |
| `SUBSCRIPTION_STATE_ON_HOLD` | On hold / suspended (auto-renewing only). User loses access. `onHoldStateContext.renewalDeclined.pendingOrderId` gives the failed order. |
| `SUBSCRIPTION_STATE_CANCELED` | Cancelled but not yet expired (auto-renewing only). All items have `autoRenewEnabled = false`. |
| `SUBSCRIPTION_STATE_EXPIRED` | All items have `expiryTime` in the past. Also the state after a revoke [11]. |
| `SUBSCRIPTION_STATE_PENDING_PURCHASE_CANCELED` | A pending transaction was cancelled. If it was for an existing subscription, use `linkedPurchaseToken` to find that subscription's current state. |

Note: the "About subscriptions" guide refers to `SUBSCRIPTION_STATE_PENDING_PURCHASE_EXPIRED` [12]. That value does not appear in the API enum [6]; trust the enum.

`canceledStateContext` says who cancelled: `userInitiatedCancellation` (with `cancelTime` and optional `cancelSurveyResult`), `systemInitiatedCancellation` (for example, a billing problem), `developerInitiatedCancellation`, or `replacementCancellation` (replaced by a plan change) [6]. `testPurchase` is present for license-tester purchases [6]. Subsist should flag these as test rows.

### Payment recovery: grace period and account hold

- **Grace period.** Enabled by default on all auto-renewing base plans [11]. It is configurable per base plan from `P0D` up to the lesser of 30 days and the billing period [23]. If it is not set, a default based on the billing period is used [23]. The per-period default values are not published in the sources reviewed (unverified). The user **keeps** access during grace, `autoRenewEnabled` stays `true`, and Google keeps extending `expiryTime` [11]. If payment recovers during grace, the renewal date does **not** reset [11]. A change to the grace length only affects subscriptions that enter grace after the change [11].
- **Silent grace period.** If the grace period is set to 0 days, Google still waits at least 1 day (24 hours) for retries. During that time the subscription stays `ACTIVE` and no grace RTDN is sent. Afterwards you receive `SUBSCRIPTION_ON_HOLD`, `SUBSCRIPTION_CANCELED`, `SUBSCRIPTION_EXPIRED` or `SUBSCRIPTION_RENEWED` [11].
- **Retry before hold.** Before entering account hold, Google makes additional charge attempts for up to 48 hours, and the user keeps benefits during that time [11].
- **Account hold.** Enabled by default for auto-renewing and installment base plans. The default length is **60 days minus the grace period** [11][21]. It is configurable from `P0D` to `P60D`, and **grace + hold must total between 30 and 60 days inclusive** [20][23]. The user **loses** access, and `expiryTime` is set in the past [11]. On recovery the purchase token stays the same, `SUBSCRIPTION_RECOVERED` is sent, and the **billing date moves to the recovery date** [11]. If hold ends unresolved, `SUBSCRIPTION_CANCELED` is sent, immediately followed by `SUBSCRIPTION_EXPIRED` [11]. During hold the user can also buy a new plan, which creates a new token and a `SUBSCRIPTION_PURCHASED` [11].
- **Subscriptions with add-ons.** The recovery period is taken from the active item with the shortest grace period [20].

### Pause

- Pause is enabled by default and can be disabled in Play Console [11]. A pause takes effect only at the end of the current billing period [11][20].
- Available pause lengths [11]: weekly subscriptions can pause for 1, 2, 3 or 4 weeks; monthly, three-month and six-month subscriptions for 1, 2 or 3 months. Google marks these lengths as "subject to change at any time". The developer guide's table also lists 1–3 months for annual subscriptions [11], but the Play Console Help Center says "annual subscriptions and free trials cannot be paused" [20]. **These sources conflict**, so treat annual pause as unverified.
- Sequence: `SUBSCRIPTION_PAUSE_SCHEDULE_CHANGED` (still `ACTIVE`), then `SUBSCRIPTION_PAUSED` (state `PAUSED`, access removed), then `SUBSCRIPTION_RECOVERED` on resume, or `SUBSCRIPTION_ON_HOLD` if the resume charge fails [11]. A manual resume moves the billing date to the resume date [11].
- PBL 8.1.0+ can return paused and on-hold subscriptions to the app with `isSuspended() = true` [1].

### Acknowledgement (3-day rule)

- A new subscription purchase that is not acknowledged **within three days** is automatically refunded and revoked [11]. Renewals do not need acknowledgement [11].
- Prepaid plans of one week or longer must be acknowledged within 3 days. Shorter plans must be acknowledged within half the plan duration (for example, 1.5 days for a 3-day plan). An unacknowledged top-up causes the whole subscription to be revoked and refunded [11][12].
- Server-side acknowledgement uses `purchases.subscriptions.acknowledge`, which is not deprecated. Since 2025-11-19 it accepts optional `externalAccountIds` [4][7][23].
- Plan changes and resubscribe are blocked while the existing subscription is still pending acknowledgement [11].
- `acknowledgementState` is `ACKNOWLEDGEMENT_STATE_PENDING` or `ACKNOWLEDGEMENT_STATE_ACKNOWLEDGED` [6]. Voided purchases can carry `voidedReason` 8, `Unacknowledged_purchase` [23].

### Upgrades, downgrades and replacement modes

An upgrade, downgrade or re-signup before expiry invalidates the old token and creates a **new purchase token** whose `linkedPurchaseToken` points at the old one [11]. Google's guidance is to revoke the entitlement attached to the linked (old) token so that two users are not entitled to the same purchase [14]. `lineItems[].itemReplacement` describes the item that was replaced, and is only available for 60 days after purchase [6].

Replacement modes (API enum `ReplacementMode`) [6][12]:

| Mode | Behavior |
|---|---|
| `WITH_TIME_PRORATION` | Immediate change. Remaining time is credited by pushing the next billing date forward. This is the default. |
| `CHARGE_PRORATED_PRICE` | Immediate upgrade. Billing date unchanged; the price difference for the rest of the period is charged. Upgrades only. |
| `CHARGE_FULL_PRICE` | Immediate change. Full price is charged now (or $0 / the intro price if the new plan has a trial or intro offer). |
| `WITHOUT_PRORATION` | Immediate change. The new price is charged at the next renewal; billing cycle unchanged. |
| `DEFERRED` | The change happens at renewal. A new purchase is issued immediately, with the old item (auto-renew off) and the new item starting after it. |
| `KEEP_EXISTING` | Item payment schedule unchanged. Used for add-on bundles (PBL 8.1.0+) [1]. |

Constraints [12]: switching to a prepaid plan only allows `CHARGE_FULL_PRICE`. Switching base plans within the same subscription to an auto-renewing plan only allows `CHARGE_FULL_PRICE` or `WITHOUT_PRORATION`. The per-base-plan default in the API is `prorationMode` = `SUBSCRIPTION_PRORATION_MODE_CHARGE_ON_NEXT_BILLING_DATE` (the default) or `SUBSCRIPTION_PRORATION_MODE_CHARGE_FULL_PRICE_IMMEDIATELY` [23]. Deferred changes appear as `lineItems[].deferredItemReplacement` / `deferredItemRemoval` [6].

### Price changes

- Changing a base plan price affects new purchases "within a few hours". Existing subscribers stay in a **legacy price cohort** until the developer ends it, either in Play Console or with `monetization.subscriptions.basePlans.migratePrices` [13]. Offer phase prices cannot be changed for existing subscribers [13].
- **Price decreases** apply at the next payment. Payment may be authorized up to 48 hours before renewal (up to 5 days in India and Brazil), so some users are charged the lower price one cycle later [13].
- **Opt-in increases (the default)**: the user must accept, or the subscription is cancelled at the first renewal at the new price. There is a 37-day advance notification period. Play stays silent for the first 7 days, then notifies users 30 days before the charge [13]. Not accepting leads to `SUBSCRIPTION_CANCELED` [11].
- **Opt-out increases**: only in some locations, with limits on amount and frequency. Users are notified 30 or 60 days ahead depending on country and are charged unless they cancel. An in-app notice is required [13].
- Installment plans: price changes apply only at the end of the commitment period [12][13].
- API fields: `lineItems[].autoRenewingPlan.priceChangeDetails` has `newPrice`, `priceChangeMode` (`PRICE_DECREASE`, `PRICE_INCREASE`, `OPT_OUT_PRICE_INCREASE`), `priceChangeState` (`OUTSTANDING`, `CONFIRMED`, `APPLIED`, `CANCELED`) and `expectedNewPriceChargeTime` [6]. `recurringPrice` excludes discounts and taxes; use `orders.get` for amounts actually paid [6].
- **Price step-up consent (South Korea)**: users must consent to the step-up after a trial or intro phase, or the subscription is cancelled. See `priceStepUpConsentDetails` (`PENDING`, `CONFIRMED`, `COMPLETED`, with `consentDeadlineTime`) and RTDN 22 [6][11].

### Refunds, revocations and voided purchases

- Refund an order with `orders.refund` (optional `revoke`; orders older than 3 years cannot be refunded) [8][23]. Revoke a subscription immediately with `purchases.subscriptionsv2.revoke`, whose `revocationContext` is one of `fullRefund`, `proratedRefund` or `itemBasedRefund` [23]. `purchases.subscriptions.refund` is deprecated; the replacement is to take `latestSuccessfulOrderId` from `subscriptionsv2.get` and call `orders.refund` [5].
- A revoke or chargeback produces `SUBSCRIPTION_REVOKED` and the state becomes `EXPIRED` [11]. Refunds also produce a `VoidedPurchaseNotification` [9].
- The Voided Purchases API only lists revoked orders. A developer refund without `revoke` does **not** appear there [15]. See [Checking current status](#checking-current-status-re-sync) for the query details.
- `Order.state` values: `PENDING`, `PROCESSED`, `CANCELED`, `PENDING_REFUND`, `PARTIALLY_REFUNDED`, `REFUNDED` [23].
- A restored subscription (Resubscribe before expiry) generates a **$0.00 confirmation order**, and billing resumes on the original date [20]. Subsist should not record this as revenue.

### Prepaid plans

- Prepaid plans do not renew and cannot be cancelled or paused. `subscriptionState` is only `ACTIVE`, `PENDING` or `CANCELED` [11].
- Every top-up issues a new purchase token with `linkedPurchaseToken` set, sends `SUBSCRIPTION_PURCHASED`, always uses `CHARGE_FULL_PRICE`, and adds time on top of the current `expiryTime`. `prepaidPlan.allowExtendAfterTime` shows when the next top-up is allowed [6][11][12].
- Durations are 1 day, 3 days, 1 week, 4 weeks, 1, 2, 3, 4, 6 or 8 months, or 1 year [21].
- Pending transactions (PBL 7+) are supported only for prepaid plans. The state goes from `PENDING` to `ACTIVE`, or to `SUBSCRIPTION_PENDING_PURCHASE_CANCELED` (RTDN 20) [12].

### Installment plans

- Available only in Brazil, France, Italy and Spain. The commitment period is 3–24 monthly payments [12][20][21].
- Here "renewal" means the end of a commitment period, but `SUBSCRIPTION_RENEWED` is sent for **each** monthly charge [11][12].
- User cancellation takes effect at the end of the commitment period, signalled by `SUBSCRIPTION_CANCELLATION_SCHEDULED` (18), then `CANCELED` and `EXPIRED` at the end [11][12]. A developer cancel with `cancellationType = DEVELOPER_REQUESTED_STOP_PAYMENTS` stops the next payment [11][23].
- `autoRenewingPlan.installmentDetails` has `initialCommittedPaymentsCount`, `subsequentCommittedPaymentsCount`, `remainingCommittedPaymentsCount` and `pendingCancellation` [6]. Missed installments are not collected, and the developer is paid per installment, not upfront [12].

### Subscriptions with add-ons, family and multi-line

- A single purchase can bundle several products (**subscriptions with add-ons**). `lineItems[]` has one entry per item, all auto-renewing or all prepaid [6][12]. Changes to items send `SUBSCRIPTION_ITEMS_CHANGED` (17) [9]. Subsist must create one ledger line per `lineItems[]` entry.
- **Family sharing / multi-line:** none of the official subscription documentation reviewed describes family sharing for Play subscriptions (unverified; treat as not applicable).

### Other

- Renewals scheduled on the 29th–31st move to the last valid day (for example 28 February) and stay on that day afterwards [11].
- A purchase token stays valid until 60 days after expiration. After that, `get` returns `410 subscriptionNoLongerAvailable` [6][11].
- Resubscribe after expiry, bought from the Play Store, creates a new token **without** `linkedPurchaseToken`. Link it to the user via `outOfAppPurchaseContext.expiredExternalAccountIdentifiers` or `expiredPurchaseToken` (present only until acknowledgement), then acknowledge it on the server [6][11].

## Checking current status (re-sync)

- **`GET https://androidpublisher.googleapis.com/androidpublisher/v3/applications/{packageName}/purchases/subscriptionsv2/tokens/{token}`** (`purchases.subscriptionsv2.get`) is the only supported way to read state. It needs only the package name and token, not the product ID [8][11]. Call it on every RTDN, at the time of the RTDN rather than at expiry time [11]. Useful fields:
  - `subscriptionState`, `startTime` and `regionCode`
  - per line item: `productId`, `expiryTime`, `latestSuccessfulOrderId`, `autoRenewingPlan.autoRenewEnabled`, `autoRenewingPlan.recurringPrice`, `offerDetails.basePlanId`, `offerDetails.offerId`, `offerDetails.offerTags`, and `offerPhase` (`freeTrial` / `introductoryPrice` / `basePrice` / `prorationPeriod`, added 2026-01-27) [4][6]
  - `etag`, which represents the current state [6]
- **Field mapping from v1.** Google's table maps the old `SubscriptionPurchase` fields to v2 [5]:
  - `orderId` becomes `lineItems.latestSuccessfulOrderId`
  - `expiryTimeMillis` becomes `lineItems.expiryTime`
  - `autoRenewing` becomes `lineItems.autoRenewingPlan.autoRenewEnabled`
  - `priceAmountMicros` and `priceCurrencyCode` become `lineItems.autoRenewingPlan.recurringPrice`
  - `countryCode` becomes `regionCode`
  - `paymentState` has no v2 equivalent. Infer it: pending payment is `PENDING`, `IN_GRACE_PERIOD` or `ON_HOLD`; received is `ACTIVE`; free trial is `lineItems.offerPhase.freeTrial`; deferred change is `lineItems.deferredItemReplacement`.
- **Amounts actually charged.** `recurringPrice` is a list price without discounts or tax. For ledger amounts, call `orders.get` or `orders.batchget` with the order IDs. `Order` has `total` (after discounts and tax), `tax` and `developerRevenueInBuyerCurrency` [6][23].
- **`linkedPurchaseToken` handling.** When a response has `linkedPurchaseToken`, the new token replaces the old one. Cases: re-signup before lapse, upgrade/downgrade, prepaid↔auto-renewing conversion, prepaid top-up [6]. Attach the new token to the same customer, mark the old token superseded, and revoke any entitlement on the old token [14]. Chains can be several links long. Resubscribe after full expiry has no link (use `outOfAppPurchaseContext`) [11].
- **User identity.** `externalAccountIdentifiers.obfuscatedExternalAccountId` and `obfuscatedExternalProfileId` are present when the app called `BillingFlowParams.Builder.setObfuscatedAccountId` / `setObfuscatedProfileId` at purchase time [6][11]. `externalAccountId` is present only if account linking happened in the purchase flow [6]. Subsist should ask customers to set `obfuscatedAccountId` to their user ID (or a hash of it) so that purchases map to `external_user_id` without extra lookup.
- **Voided purchases backfill.** Call `GET .../applications/{packageName}/purchases/voidedpurchases` with `type=1` to include subscriptions [15][16]:
  - Once subscriptions are requested, key rows by `orderId`, not `purchaseToken` [15].
  - The window covers only the past 30 days, and filtering uses the time Google marked the order voided, not `voidedTimeMillis` [15].
  - `maxResults` defaults to and caps at 1000; paginate with `token` / `nextPageToken` [15].
  - Quota: 6000 queries per day and 30 per 30 seconds, per package [15].
  - Each row has `voidedSource` (0 User, 1 Developer, 2 Google) and `voidedReason` (0 Other, 1 Remorse, 2 Not_received, 3 Defective, 4 Accidental_purchase, 5 Fraud, 6 Friendly_fraud, 7 Chargeback, 8 Unacknowledged_purchase) [23].
  - Run it daily as a safety net for missed RTDNs.
- **Error handling** [6]:
  - `410 purchaseTokenNoLongerValid` or `subscriptionNoLongerAvailable`: stop polling that token.
  - `409 concurrentUpdate`: retry with backoff.
  - `403 userInsufficientPermission`: surface a credentials problem to the customer.

## Migration notes for subsist-app

> **Status (2026-09-28):** implemented in `subsist-app` (not yet deployed). The package is now `google_play/` (app label still `google`). `client.py` uses a service account against Developer API v3; `sync.py` reads `purchases.subscriptionsv2.get`, charged amounts from `orders.batchget`, follows the `linkedPurchaseToken` chain and optionally acknowledges; `views.py` receives authenticated Pub/Sub pushes at `/google-play/notifications/<app id>/`, deduplicates on `messageId` and maps events per the table above; `google_play_voided` backfills refunds daily. Still open: encrypting the stored service-account key, `PendingRefundReviewNotification` responses, and add-on (multi-line-item) bundles beyond the first line item's receipt fields.

What `/Users/johnromanski/Digital Tractor/subsist/subsist-app/google/models.py` does today, and what it should do instead:

- **API version.** It builds `build("androidpublisher", "v2", ...)`. v2 was shut down on 2019-12-01 [22], so every call fails today. Use `androidpublisher` **v3**.
- **Endpoint.** It calls `purchases().subscriptions().get(packageName, subscriptionId, token)`. That method has been deprecated since 2025-05-21, shuts down 2027-08-31 [5], and is already missing from the current discovery document and from client libraries released after 2026-07-01 [5][23]. Switch to `purchases().subscriptionsv2().get(packageName=..., token=...)`, which needs no product ID [8].
- **Credentials.** It uses the OAuth web-client trio (`client_id`, `client_secret`, `refresh_token`) against `https://accounts.google.com/o/oauth2/token` [19]. Replace this with a service account: `google.oauth2.service_account.Credentials.from_service_account_info(info, scopes=["https://www.googleapis.com/auth/androidpublisher"])`, with the service account invited in Play Console as described in [Credentials required](#credentials-required) [18]. Store the JSON key encrypted per customer and per app.
- **Credential bug.** `_service()` passes the literal strings `"client_id"`, `"client_secret"` and `"refresh_token"` instead of `self.client_id` and the other attributes, so it could never authenticate. This goes away with service accounts.
- **Response parsing bug.** `json.loads(validate_request.execute())` is wrong because `execute()` already returns a dict. Also, `linked_receipt` is undefined when there is no `linkedPurchaseToken` (NameError). And `from_timestamp` calls `datetime.datetime.fromtimestamp` although `datetime` is imported as the class. Rewrite these rather than patch them.
- **Order and subscription identity.** It derives `subscription_id` by `orderId.rsplit("..")`. The initial order has no `..` suffix, so the two-value unpack raises `ValueError`, and Google does not document the order ID format as a contract (unverified). Use the **purchase token chain** (the token plus its `linkedPurchaseToken` ancestors) as the subscription identity, and `lineItems[].latestSuccessfulOrderId` as the transaction ID [5][6].
- **Price.** It reads `priceAmountMicros` / `priceCurrencyCode` (list price, v1). Read the charged amount per order from `orders.get` (`total`, `tax`, `developerRevenueInBuyerCurrency`), and keep `autoRenewingPlan.recurringPrice` only as the list price [6][23].
- **Dates.** It uses `startTimeMillis` / `expiryTimeMillis` (epoch ms). Use `startTime` and per-line-item `expiryTime`, which are RFC 3339 strings [6].
- **Renewal and payment status.** It maps `autoRenewing` to renewal status and `paymentState` 0–3 to payment status. `paymentState` has no v2 equivalent [5]. Derive status from `subscriptionState` and `autoRenewingPlan.autoRenewEnabled`; detect trials via `lineItems[].offerPhase.freeTrial` and deferred changes via `deferredItemReplacement` [5][6].
- **Receipt model.** `GoogleReceipt` stores one `product_id`. v2 purchases can have multiple `lineItems` (add-ons) [6]. Store `base_plan_id`, `offer_id` and `offer_tags` per line item, plus `subscription_state`, `acknowledgement_state`, `region_code`, `obfuscated_external_account_id`, `test_purchase` and the raw v2 response.
- **Linked tokens.** `linked_purchase_token` is a `ForeignKey(..., on_delete=PROTECT)` that is only resolved if the old receipt already exists locally. Store the linked token as a plain string, backfill the FK when possible, and mark the old token superseded [11][14].
- **Event ingestion.** There is no RTDN handling at all; the code only fetches on demand. Add a Pub/Sub push endpoint (or a pull worker) that decodes `message.data`, de-duplicates on `messageId`, calls `subscriptionsv2.get`, and emits events from the mapping table above [9][10].
- **Voided purchases.** Not handled. Add a daily `voidedpurchases.list?type=1` job keyed by `orderId` [15], plus `VoidedPurchaseNotification` handling [9], to emit `refund.issued` and `revoked`.
- **Acknowledgement.** Not handled. If Subsist is the customer's backend of record, call `purchases.subscriptions.acknowledge` within 3 days of each new token (including top-ups and plan changes), or confirm that the customer's app acknowledges. Unacknowledged purchases are auto-refunded [7][11].
- **Transaction deduplication.** The code uses `Transaction.objects.get_or_create` on every field, including mutable ones like `renewal_status`, which creates duplicates whenever state changes. Key transactions on `(merchant, order_id)` and update the mutable fields.

## References

1. Google Play Billing Library release notes — https://developer.android.com/google/play/billing/release-notes (accessed 2026-09-28)
2. Google Play Billing Library version deprecation — https://developer.android.com/google/play/billing/deprecation-faq (accessed 2026-09-28)
3. Migrate to Google Play Billing Library 9 from versions 7 or 8 — https://developer.android.com/google/play/billing/migrate-gpblv9 (accessed 2026-09-28)
4. Google Play Developer API release notes — https://developer.android.com/google/play/billing/play-developer-apis-release-notes (accessed 2026-09-28)
5. Deprecations (Google Play Developer APIs) — https://developer.android.com/google/play/billing/play-developer-apis-deprecations (accessed 2026-09-28)
6. REST Resource: purchases.subscriptionsv2 — https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.subscriptionsv2 (accessed 2026-09-28)
7. REST Resource: purchases.subscriptions — https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.subscriptions (accessed 2026-09-28)
8. Google Play Android Developer API (REST reference index) — https://developers.google.com/android-publisher/api-ref/rest (accessed 2026-09-28)
9. Real-time developer notifications reference guide — https://developer.android.com/google/play/billing/rtdn-reference (accessed 2026-09-28)
10. Getting ready (Play Billing) — https://developer.android.com/google/play/billing/getting-ready (accessed 2026-09-28)
11. Subscription lifecycle — https://developer.android.com/google/play/billing/lifecycle/subscriptions (accessed 2026-09-28)
12. About subscriptions — https://developer.android.com/google/play/billing/subscriptions (accessed 2026-09-28)
13. Change subscription prices — https://developer.android.com/google/play/billing/price-changes (accessed 2026-09-28)
14. Fight fraud and abuse — https://developer.android.com/google/play/billing/security (accessed 2026-09-28)
15. Voided Purchases API — https://developers.google.com/android-publisher/voided-purchases (accessed 2026-09-28)
16. Method: purchases.voidedpurchases.list — https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.voidedpurchases/list (accessed 2026-09-28)
17. REST Resource: orders — https://developers.google.com/android-publisher/api-ref/rest/v3/orders (accessed 2026-09-28)
18. Google Play Developer API: Getting Started — https://developers.google.com/android-publisher/getting_started (accessed 2026-09-28)
19. Google Play Developer API: Authorization — https://developers.google.com/android-publisher/authorization (accessed 2026-09-28)
20. Understanding subscriptions (Play Console Help) — https://support.google.com/googleplay/android-developer/answer/12154973 (accessed 2026-09-28)
21. Create and manage subscriptions (Play Console Help) — https://support.google.com/googleplay/android-developer/answer/140504 (accessed 2026-09-28)
22. Changes to the Google Play Developer API (Android Developers Blog, March 2019) — https://android-developers.googleblog.com/2019/03/changes-to-google-play-developer-api.html (accessed 2026-09-28)
23. Google Play Android Developer API v3 discovery document (revision 20260928; schemas `VoidedPurchase`, `AutoRenewingBasePlanType`, `Order`, `RevocationContext`, `CancellationContext`, `SubscriptionPurchasesAcknowledgeRequest`) — https://androidpublisher.googleapis.com/$discovery/rest?version=v3 (accessed 2026-09-28)
