# Apple App Store

> Last reviewed: 2026-09-28 · Current versions: **App Store Server API 1.21** (2026-04-27; server update 2026-05-05 moved the recommended domain to `api.storekit.apple.com`) [2] · **App Store Server Notifications V2** (payload `version` "2.0"; latest changelog entry 2026-04-27) [4][7] · **Retention Messaging API 1.5** (2026-04-27, pre-release) [51] · **Advanced Commerce API 1.2** (2025-12-10; latest server update 2026-04-22) [53] · **App Store Server Library** Python v3.1.2 (2026-06-01), Node v3.1.0, Java v5.2.0, Swift v6.0.0 [58] · `verifyReceipt` and Notifications V1 **deprecated** since 2023-06-05 [4][12]

## Overview

Apple sells auto-renewable subscriptions through In-App Purchase. Every subscription product belongs to a **subscription group**, and a customer can hold only one subscription per group at a time; a group can contain up to 100 subscriptions [42]. Inside a group you rank products into **levels**, from level 1 (the most content) downward. The level ranking decides whether a change is an upgrade, a downgrade, or a crossgrade, and products with the same content but different durations can share a level [41][42]. Durations are 1 week, 1, 2, 3, or 6 months, or 1 year, and a subscription renews on the same calendar day as the original purchase (clamped to the last day of shorter months) [41]. Since April 2026, a yearly product can also offer a **monthly billing plan with a 12-month commitment** (outside the United States and Singapore). It keeps the same product ID and shows up as `billingPlanType = MONTHLY` in transactions and renewal info [54][60].

On the server side, Apple identifies a subscription by its **`originalTransactionId`**, which stays the same across renewals. Each billing event creates a new `transactionId` [26][37]. Apple pushes lifecycle events to your server as signed **App Store Server Notifications V2**, and you can pull current state at any time from the **App Store Server API**. Both deliver the same JWS-signed transaction (`JWSTransactionDecodedPayload`) and renewal-info (`JWSRenewalInfoDecodedPayload`) objects [1][3][8]. Apple states that the transaction `price`/`currency` fields must not be used for revenue reconciliation or recognition; App Store Connect reporting is the source of record for finance [26].

## APIs and versions

| API | Current version | What Subsist uses it for | Status |
|---|---|---|---|
| App Store Server API (`https://api.storekit.apple.com/inApps/...`, sandbox `https://api.storekit-sandbox.apple.com/`) | 1.21 (2026-04-27) [2] | Pull current subscription status, full transaction history, single transactions, refund history, notification history/replay, test notifications [1] | Current. Old `*.itunes.apple.com` domains are still supported after the 2026-05-05 domain change [2] |
| App Store Server Notifications V2 | V2 (`version` "2.0"); last changed 2026-04-27 [4][7] | Real-time push of lifecycle events into the ledger | Current [3] |
| App Store Server Notifications V1 (`notification_type`, `responseBodyV1`) | — | Not used | **Deprecated** 2023-06-05 [4] |
| `verifyReceipt` (`buy.itunes.apple.com` / `sandbox.itunes.apple.com`) | — | Legacy only (what `subsist-app` uses today) | **Deprecated.** Docs say the deprecation date is in the HTTP header (RFC 8594) [12]. On 2026-09-28 the endpoint still answered, with header `Deprecation: Mon, 5 Jun 2023 23:59:59 GMT`. No sunset/removal date was published (observed live; no removal date found in docs) [13] |
| Get Transaction History **V1** (`/inApps/v1/history`) | — | Not used | **Deprecated** in API 1.12 (2024-06-10); use Get Transaction History (`/inApps/v2/history`) [2] |
| Get Refund History **V1** | — | Not used | **Deprecated** in API 1.6 (2022-08-08); use `/inApps/v2/refund/lookup` [2] |
| StoreKit 2 (Swift "Apple In-App Purchase" API, `Transaction`, `appAccountToken`) | Ships with the OS | Client side, owned by the Subsist customer's app. Relevant to Subsist because it sets `appAccountToken` [36][59] | Current |
| Original StoreKit API (`SKPaymentQueue`, receipts) | — | Not used | Deprecated (e.g. `SKPaymentQueue` deprecated in iOS 18.0 / macOS 15.0) [55][56] |
| App Store Server Library (Swift, Java, Python, Node) | Python v3.1.2 (2026-06-01) [58] | Recommended: JWT creation, API client, JWS verification (`verifyAndDecodeTransaction`, `verifyAndDecodeRenewalInfo`), receipt → transactionId extraction for migration [16] | Current |
| Retention Messaging API | 1.5 (2026-04-27) [51] | Optional, not needed for the ledger. Lets developers show a message or offer when a customer taps Cancel [50] | Pre-release; access by request [50] |
| Advanced Commerce API | 1.2 (2025-12-10) [53] | Only for customers with large custom SKU catalogs. Its transactions use the same JWS formats and emit `PRICE_CHANGE`, `METADATA_UPDATE`, `MIGRATION` notifications [5][52] | Current; access by application [52] |

## Credentials required

A Subsist customer must provide the following:

| Item | What it is | Where to get it |
|---|---|---|
| In-App Purchase private key (`.p8`) | ES256 private key that signs API JWTs. It works for the App Store Server API, Advanced Commerce API, and External Purchase Server API only [14] | App Store Connect → Users and Access → **Integrations** tab → Keys: **In-App Purchase** → Generate In-App Purchase Key. The file **can be downloaded only once**, and Apple keeps no copy [14] |
| Key ID | The `kid` of that key (e.g. `2X9R4HXF34`) [15] | Same page, listed next to the active key ("Copy Key ID") [15] |
| Issuer ID | UUID for the team, used as `iss` [15] | Users and Access → Keys page, near the top ("Copy") [15] |
| Bundle ID | App's bundle identifier, used as `bid` claim and for JWS verification [15][16] | App Store Connect → App Information (must match Xcode) [57] |
| App Apple ID (`appAppleId`) | Numeric app identifier. The App Store Server Library's `SignedDataVerifier` requires it for the Production environment [58]. It is absent from sandbox notifications [8] | App Store Connect → App Information → Apple ID (read-only) [57] |
| Environment(s) | `Production` and/or `Sandbox`. A transaction ID must be queried in the environment that created it [1] | Decide per app. Subsist should support both |
| Notification URL configured | Subsist's webhook URL entered as a **Version 2** URL | See [Server notifications](#server-notifications) [11] |

The legacy app-specific **shared secret** (`password` field of `verifyReceipt`) is **not** needed for the App Store Server API.

**Request authentication.** Each API call carries `Authorization: Bearer <JWT>` [15]. The JWT is built as follows:
- Header: `alg: ES256`, `kid: <Key ID>`, `typ: JWT`.
- Payload: `iss` (Issuer ID), `iat` (UNIX seconds), `exp` (no more than 60 minutes after `iat`), `aud: "appstoreconnect-v1"`, `bid` (bundle ID).
- Signature: signed with the `.p8` key.

Apple says to generate a new token for each request or reuse it until it expires [1][15]. The server must support TLS 1.2 or later [1]. Rate limits are per app, enforced hourly, and expressed per second: for example 50/s for Get All Subscription Statuses, Get Transaction History, Get Transaction Info, and Get Notification History; 10/s for Get Refund History; and 1/s for Request a Test Notification. Sandbox limits are 10% of those. When a limit is exceeded, the API returns HTTP 429 with a `Retry-After` header (UNIX ms) [23].

## Server notifications

**Setup.**
1. The Subsist customer enters Subsist's HTTPS URL in App Store Connect → the app → General → **App Information** → App Store Server Notifications. There is a **Production Server URL** and a **Sandbox Server URL**; choose **Version 2** for each [11].
   - If no sandbox URL is set, sandbox notifications also go to the production URL. If only a sandbox URL is set, no production notifications are sent [11].
   - Allowed ports are 443 or ≥1024. TLS 1.2 or later is required. For IP allow-lists, use `17.0.0.0/8` [9].
2. Each POST body (`responseBodyV2`) contains a `signedPayload` JWS [3]. Once decoded (`responseBodyV2DecodedPayload`), it contains `notificationType`, `subtype`, `notificationUUID` (use it to deduplicate), `signedDate`, `version`, and exactly one of `data`, `summary`, `externalPurchaseToken`, or `appData` [7].
   - `data` contains `appAppleId`, `bundleId`, `bundleVersion`, `environment`, `status` (subscription status as of `signedDate`), `signedTransactionInfo`, and `signedRenewalInfo` (subscriptions only). Both signed fields are themselves JWS [8].
   - Verify every JWS against Apple's root certificates. The App Store Server Library does this for you [16].
3. **Respond** with HTTP 200–206 for success. For failure, return 40x/50x, and Apple will retry. In **production**, V2 notifications are retried 5 times, at 1, 12, 24, 48, and 72 hours after the previous attempt. **Sandbox notifications are sent only once** [10].
4. **Test endpoint.** `POST /inApps/v1/notifications/test` asks Apple to send a `TEST` notification to the URL for that environment. The response includes a `testNotificationToken`; pass it to Get Test Notification Status to see the delivery result [21].
5. **Notification history (replay).** `POST /inApps/v1/notifications/history` returns V2 notifications Apple attempted to send over the last **180 days (production)** or **30 days (sandbox)** [20].
   - The request can filter by `notificationType`/`notificationSubtype` or by a single `transactionId`, and `onlyFailures` limits results to failed deliveries [2][20].
   - Results come in pages of 20, using `paginationToken` / `hasMore` [20].
   - History records show the state at the time the notification was sent. Use Get All Subscription Statuses for current state [20].

**Notification types.** The table below lists every `notificationType` value copied from Apple's `notificationType` and `subtype` reference [5][6]. "Subsist event" is the proposed mapping.

| notificationType | subtype | Meaning (per Apple) | Subsist event |
|---|---|---|---|
| `SUBSCRIBED` | `INITIAL_BUY` | First purchase in the group, or first access via Family Sharing. Also sent when an offer code is used to subscribe for the first time [5] | `subscription.started`. Emit `trial.started` instead when the transaction has `offerType` 1 and `offerDiscountType` `FREE_TRIAL`. Also emit `offer.redeemed` when `offerType` is 2, 3, or 4 [28][29] |
| `SUBSCRIBED` | `RESUBSCRIBE` | Resubscribed (or got Family Sharing access) to the same or another product in the group after it expired. Also sent for a promotional, offer-code, or **win-back** redemption after expiry [5] | `subscription.started` (plus `offer.redeemed` if `offerType` is present) |
| `DID_RENEW` | *(none)* | Active subscription auto-renewed for a new period [5] | `payment.renewed`. Also emit `trial.converted` if the previous period was a free trial (Subsist derives this; Apple sends no separate conversion event) |
| `DID_RENEW` | `BILLING_RECOVERY` | An expired subscription that had failed to renew has now renewed [5][6] | `payment.renewed` (recovery from retry or grace) |
| `DID_FAIL_TO_RENEW` | *(none)* | Renewal failed because of billing; the subscription is in billing retry with no grace period, so service may stop [5] | `payment.retrying` |
| `DID_FAIL_TO_RENEW` | `GRACE_PERIOD` | Renewal failed and the subscription is in a Billing Grace Period; keep providing service [5] | `grace_period.started` (retry also runs) |
| `GRACE_PERIOD_EXPIRED` | *(none)* | Grace period ended without renewal. Billing retry continues [5] | `payment.retrying` (grace over; entitlement off) |
| `DID_CHANGE_RENEWAL_STATUS` | `AUTO_RENEW_DISABLED` | Customer turned off auto-renew, or Apple turned it off after the customer requested a refund [5][6] | `renewal.disabled` |
| `DID_CHANGE_RENEWAL_STATUS` | `AUTO_RENEW_ENABLED` | Customer re-enabled auto-renew [6] | `renewal.enabled` |
| `DID_CHANGE_RENEWAL_STATUS` | *(none)* | Listed in Apple's price-increase table for "customer canceled after receiving a price increase notice or consent request" [5] | `renewal.disabled` (check `autoRenewStatus`) |
| `DID_CHANGE_RENEWAL_PREF` | `UPGRADE` | Upgrade (or crossgrade with the same duration). Takes effect **immediately** and starts a new billing period, with a prorated refund of the old one [5][6] | `plan.changed` (effective now) |
| `DID_CHANGE_RENEWAL_PREF` | `DOWNGRADE` | Downgrade (or crossgrade to a different duration). Takes effect at the **next renewal** [5][6] | `plan.changed` (scheduled; effective at `renewalDate`) |
| `DID_CHANGE_RENEWAL_PREF` | *(none)* | Customer switched back to the current product, which cancels a pending downgrade [5] | `plan.changed` (pending change cancelled) |
| `OFFER_REDEEMED` | *(none)* | Customer with an **active** subscription redeemed a promotional offer or offer code [5] | `offer.redeemed` |
| `OFFER_REDEEMED` | `UPGRADE` | Offer redeemed as an upgrade, effective immediately [5] | `offer.redeemed` + `plan.changed` |
| `OFFER_REDEEMED` | `DOWNGRADE` | Offer redeemed as a downgrade, effective at next renewal [5] | `offer.redeemed` + `plan.changed` (scheduled) |
| `PRICE_INCREASE` | `PENDING` | Customer was told about a price increase that needs consent and hasn't responded [5][6] | `price.changed` (pending consent) |
| `PRICE_INCREASE` | `ACCEPTED` | Customer consented, **or** the increase doesn't need consent and the customer was notified [5][6] | `price.changed` |
| `EXPIRED` | `VOLUNTARY` | Expired after the customer turned off auto-renew [6] | `subscription.expired` |
| `EXPIRED` | `BILLING_RETRY` | Billing retry period ended without a successful charge [6] | `subscription.expired` |
| `EXPIRED` | `PRICE_INCREASE` | Customer didn't consent to a price increase that required consent [6] | `subscription.expired` |
| `EXPIRED` | `PRODUCT_NOT_FOR_SALE` | Product wasn't for sale at renewal time [6] | `subscription.expired` |
| `EXPIRED` | *(none)* | Expired for some other reason [5] | `subscription.expired` |
| `REFUND` | *(none)* | Apple refunded a transaction. `revocationDate` and `revocationReason` are set, and since API 1.19 `revocationType` (`REFUND_FULL` / `REFUND_PRORATED`) and `revocationPercentage` are set too [5][26][32] | `refund.issued` |
| `REFUND_REVERSED` | *(none)* | Apple reversed a prior refund after a customer dispute. Reinstate access; the renewal date is unchanged [5] | — (reversal of `refund.issued`; not in the vocabulary) |
| `REFUND_DECLINED` | *(none)* | Apple declined a refund the customer requested through the in-app refund API [5] | — |
| `CONSUMPTION_REQUEST` | *(none)* | Customer requested a refund; Apple asks for consumption data. Respond within **12 hours** via Send Consumption Information, and only if the customer consented [5][49] | — |
| `REVOKE` | *(none)* | A Family Sharing member lost access (purchaser stopped sharing, someone left the family, or the purchaser was refunded) [5] | `revoked` |
| `RENEWAL_EXTENDED` | *(none)* | Apple extended the renewal date of one subscription at the developer's request [5] | — (update period end) |
| `RENEWAL_EXTENSION` | `SUMMARY` | A mass renewal-date extension finished (payload has `summary`, not `data`) [5][7] | — |
| `RENEWAL_EXTENSION` | `FAILURE` | The mass extension failed for one subscription [5] | — |
| `ONE_TIME_CHARGE` | *(none)* | Consumable, non-consumable, or non-renewing purchase, including offer codes and Family Sharing of non-consumables. Not auto-renewable subscriptions [5] | — (out of scope for subscriptions) |
| `TEST` | *(none)* | Sent only when requested through Request a Test Notification [5] | — (health check) |
| `EXTERNAL_PURCHASE_TOKEN` | `CREATED`, `ACTIVE_TOKEN_REMINDER`, `UNREPORTED` | External Purchase API (alternative payments) only [5][6] | — |
| `RESCIND_CONSENT` | *(none)* | Parent or guardian withdrew consent for a child's app use. Payload has `appData` [5][7] | — |
| `PRICE_CHANGE` | *(none)* | Advanced Commerce API only: the developer called Change Subscription Price [5] | `price.changed` (ACA only) |
| `METADATA_UPDATE` | *(none)* | Advanced Commerce API only: Change Subscription Metadata was called [5] | — |
| `MIGRATION` | *(none)* | Advanced Commerce API only: a subscription was migrated to ACA [5] | — |

Notes on this list:
- The changelog entry for 2025-03-24 calls the ACA migration type `MIGRATE`, but the `notificationType` reference page lists `MIGRATION` [4][5]. Subsist should accept both strings and log unknown types instead of failing.
- `subscription.paused` has no Apple equivalent. Apple's subscription `status` values are only 1 = active, 2 = expired, 3 = billing retry, 4 = billing grace period, and 5 = revoked [24].

## Lifecycle rules and edge cases

- **Billing retry.** When a renewal fails for billing reasons, Apple retries for **up to 60 days** [5][38].
  - Recovery **during grace** keeps the original billing cycle, with no loss of paid days [38][39].
  - Recovery **after grace, but within retry,** starts a new billing date on the recovery date [38].
  - Renewal info shows `isInBillingRetryPeriod`, and `status` = 3 [24][27].
- **Billing Grace Period.** This is an app-level opt-in in App Store Connect with a choice of **3, 16, or 28 days** [39].
  - Weekly subscriptions get 3 days (3-day setting) or **6 days** (16- or 28-day setting). Monthly and yearly subscriptions get the full 3, 16, or 28 days [39].
  - It can apply to "All Renewals" (including free-to-paid) or "Only Paid to Paid Renewals", and can be enabled per environment [39].
  - Edits take up to 24 hours and affect only upcoming renewals [39].
  - It **does not apply to monthly-with-12-month-commitment plans** [39].
  - Renewal info shows `gracePeriodExpiresDate`, and `status` = 4 [24][27].
- **Expiration intent** (`expirationIntent` in renewal info) [25]:
  - 1 = customer canceled.
  - 2 = billing error.
  - 3 = didn't consent to a price increase or to an offer conversion that needed consent.
  - 4 = product not for sale.
  - 5 = other.
- **Upgrades, downgrades, and crossgrades** [41][30]:
  - Upgrade (to a higher level) is immediate: the old plan is prorated and refunded to the original payment method, and the upgrade date becomes the new renewal date.
  - Downgrade takes effect at the next renewal.
  - Crossgrade (same level) is immediate if the duration is the same, and waits until the next renewal if the duration differs.
  - An upgrade's transaction has `transactionReason` = `PURCHASE`, and the replaced transaction gets `isUpgraded` = true. A downgrade shows up as a `RENEWAL` transaction on the renewal date [26][30].
  - For 12-month commitment plans, upgrades end the commitment immediately, and downgrades wait until the end of the 12-month commitment [54].
- **Price increases** [40]:
  - Subscribers must **consent** when any of the following is true: the storefront requires consent for any increase; the increase is more than 50% **and** more than about US$5 per period (non-annual) or US$50 per year (annual) (per-storefront thresholds are listed in [61]); or the subscriber already had an increase on that subscription within the past 12 months.
  - Otherwise, Apple only **notifies** the subscriber.
  - Increases that fall within the minimum notice period (7 days for weekly, 27 days for monthly, 30 days for longer durations) push the new price out by one more period. Subscribers on intro or promotional pricing renew at least once more at the old price.
  - A subscriber who doesn't consent expires at the end of the last period at the old price (`EXPIRED`/`PRICE_INCREASE`, `expirationIntent` 3).
  - A developer can preserve the current price for existing subscribers. Preserved-price subscribers who lapse can resubscribe at that price within 60 days.
  - Price **decreases** apply automatically to existing subscribers and can't be reversed [40].
  - Renewal info reports `priceIncreaseStatus` (0 = no response yet to an increase that needs consent, 1 = consented or notified) and `renewalPrice`/`currency` (milliunits) [27][35].
- **Refunds and revocation.**
  - A refund arrives as `REFUND`. The transaction gets `revocationDate` and `revocationReason` (0 = other, e.g. accidental purchase; 1 = issue with the app) [31], plus `revocationType` and `revocationPercentage` (0–100000 milliunits) for prorated refunds [26][32]. Subscription `status` becomes 5 [24].
  - Customer refund requests produce `DID_CHANGE_RENEWAL_STATUS`/`AUTO_RENEW_DISABLED` (Apple turns off auto-renew), `CONSUMPTION_REQUEST`, and then either `REFUND` or `REFUND_DECLINED`. `REFUND_REVERSED` can follow a refund [5].
  - Get Refund History (`/inApps/v2/refund/lookup/{id}`, 20 per page, `revision`) backfills refunds that were missed [22].
- **Family Sharing.**
  - The developer opts in per product, and **once turned on it can't be turned off** [46]. The purchaser can share with up to five family members [45][46].
  - Each member gets their own transactions with `inAppOwnershipType` = `FAMILY_SHARED` (the purchaser's is `PURCHASED`) [33].
  - A member gaining access produces `SUBSCRIBED` (`INITIAL_BUY`/`RESUBSCRIBE`), and losing it produces `REVOKE` (`revocationReason` 0 if they left the family or sharing stopped; it matches the purchaser's reason if the purchaser was refunded) [5][31].
  - Subsist should not count `FAMILY_SHARED` transactions as revenue.
  - Set App Account Token doesn't work on family-shared transactions (`FamilyTransactionNotSupportedError`) [2].
- **Offers.** `offerType` values: 1 = introductory, 2 = promotional, 3 = offer code, 4 = win-back. All types except 1 carry an `offerIdentifier` [28]. `offerDiscountType` values are `FREE_TRIAL`, `PAY_AS_YOU_GO`, `PAY_UP_FRONT`, and `ONE_TIME` (non-subscription offer codes, added 2025-10-29) [29]. `offerPeriod` gives the offer duration (added in API 1.15) [2][26].
  - **Offer codes** are redeemable in the App Store, through a URL, or in the app. Limits: up to 10 active offers and 1,000,000 codes per app per quarter. Redemption produces `OFFER_REDEEMED` on an active subscription, `SUBSCRIBED` for a new or lapsed subscriber (behavior changed 2024-01-23), or `ONE_TIME_CHARGE` for non-subscription products [4][43].
  - **Win-back offers** target churned subscribers (expired, auto-renew off). Eligible offer IDs are listed in renewal info as `eligibleWinBackOfferIds`, best first. By default, redemption can complete outside the app ("Streamlined Purchasing") [44].
- **Ask to Buy.** A child's purchase is **pending** until a parent approves it. StoreKit returns `Product.PurchaseResult.pending`, and the purchase only succeeds after approval [47]. Subsist should expect no transaction or notification for a declined or pending request (inferred from [47]; Apple's server docs don't describe a separate server event).
- **Sandbox vs production.**
  - `environment` is set on every transaction, renewal info, and notification [8][26]. Call the API in the environment that created the transaction ID. If the environment is unknown, try production first and fall back to sandbox on error `4040010` [1].
  - Sandbox renewals are accelerated: by default 1 month = 5 minutes (1 week = 3 min, 1 year = 1 hour). A subscription auto-renews up to 12 times, and auto-renewal turns off on the 13th attempt. The renewal-rate setting also shortens billing retry and grace in sandbox [48].
  - Sandbox notifications are not retried [10]. Sandbox notification history covers 30 days [20]. Sandbox rate limits are 10% of production [23]. `appAppleId` is absent in sandbox [8].
- **Monthly with 12-month commitment** (new in 2026): each month produces its own transaction. Canceling stops the next commitment renewal but not the remaining months. `commitmentInfo` / `billingPlanType` appear on transactions, and `renewalBillingPlanType` on renewal info [2][54].

## Checking current status (re-sync)

- **Get All Subscription Statuses**: `GET /inApps/v1/subscriptions/{anyTransactionId}` [17].
  - Returns every auto-renewable subscription the customer has in the app, grouped by `subscriptionGroupIdentifier`. Each `lastTransactionsItem` has `originalTransactionId`, `status` (1–5), `signedTransactionInfo`, and `signedRenewalInfo` [17][24].
  - Optional repeated filter `status=` (e.g. `?status=1&status=4`). The call is not paginated.
  - This is the authoritative way to get current state after an outage, or to reconcile a subscriber [10][17].
- **Get Transaction History**: `GET /inApps/v2/history/{anyTransactionId}` [18].
  - Returns every transaction of every product type in any state, including refunded, revoked, and finished transactions.
  - Paging: **20 per page**. Keep calling with the returned `revision` token while `hasMore` is true, and repeat the same query parameters on each call [18][63].
  - Filters: `startDate`, `endDate`, `productId`, `productType`, `inAppOwnershipType`, `subscriptionGroupIdentifier`, `revoked`, and `sort` (`ASCENDING` by default, ordered by last-modified date). With ascending order, an updated transaction can appear again with new data [18].
- **Get Transaction Info**: `GET /inApps/v1/transactions/{transactionId}` returns one signed transaction [19].
- **Get Refund History**: `GET /inApps/v2/refund/lookup/{anyTransactionId}`, 20 per page with `revision` [22].
- **Get Notification History**: replays missed notifications (180 days in production) [20].
- **Identifying a subscriber.**
  - `anyTransactionId` in the paths above accepts an `originalTransactionId`, a `transactionId`, or an `appTransactionId` [17][18].
  - Store **`originalTransactionId`** as the subscription key. Apple recommends saving it to uniquely identify auto-renewable subscriptions [37].
  - To link a subscription to the Subsist customer's own user, rely on **`appAccountToken`**. This is a UUID the app passes at purchase (`appAccountToken(_:)` in StoreKit 2, or `applicationUsername` in the original API), and Apple echoes it in transactions and renewal info [36].
  - Since API 1.16 (2025-06-09), the server can set or change it with **Set App Account Token**, for example for purchases made outside the app. It works only on original transaction IDs and not on family-shared transactions [2][36].
  - `appTransactionId` (added 2025-02-21) identifies the customer's app download and is present on transactions and renewal info [2][26].

## Migration notes for subsist-app

> **Status (2026-09-28):** implemented in `subsist-app` (not yet deployed). `apple/client.py` wraps the App Store Server Library; `apple/sync.py` loads Get All Subscription Statuses and Get Transaction History (v2); `apple/views.py` receives Notifications V2 at `/apple/notifications/<app id>/` and deduplicates on `notificationUUID`; `apple/events.py` maps notifications to Subsist events per the table above; `manage.py apple_convert_receipts` converts stored legacy receipts. Still open: encrypting the stored `.p8` key, replaying missed notifications with Get Notification History, and Family Sharing / price-increase fields beyond what is stored in `merchant_response`.

What `subsist-app/apple/models.py` does today, compared with what it should do:

- **Endpoint.** It POSTs `receipt-data` + `password` (shared secret) to `buy.itunes.apple.com/verifyReceipt`, falling back to sandbox on status `21007`. That endpoint has been **deprecated since 2023-06-05** [12]. Replace it with App Store Server API calls authenticated by an ES256 JWT (In-App Purchase key, Key ID, Issuer ID, bundle ID) [15]. Use the App Store Server Library for Python (`pip install app-store-server-library`, v3.1.2) instead of hand-rolled `requests` code [16][58].
- **Bug: sandbox fallback.** On `21007` it calls `self.fetch_subscription_data(production=False)`, which does not exist, so any sandbox receipt raises `AttributeError`. The new flow picks the environment from the notification or transaction `environment` field, or tries production then sandbox on error `4040010` [1].
- **Converting existing receipts.** For each stored `AppleReceipt.receipt`, use the library's `ReceiptUtility.extract_transaction_id_from_app_receipt` to get a transaction ID [16][58]. Then call Get All Subscription Statuses and Get Transaction History (v2) once to backfill the ledger, and store `originalTransactionId`.
- **Push instead of poll.** The current code has no webhook. Add a V2 notification endpoint that verifies `signedPayload` with `SignedDataVerifier` (needs Apple root certs, bundle ID, and `appAppleId` in production) [16][58]. Deduplicate on `notificationUUID` [7], return 200 quickly [10], and backfill with Get Notification History [20].
- **Transaction identity.** It uses `web_order_line_item_id` as `transaction_id`. Use `transactionId` (unique per billing event) as the ledger transaction key, and `originalTransactionId` as `subscription_id` [26][37]. `webOrderLineItemId` is still available on JWS transactions if needed [26].
- **Amounts.** It writes `currency_code="n/a"` and `payment_amount="n/a"`. JWS transactions carry `price` (milliunits, int64) and `currency` (ISO 4217) since API 1.10, and renewal info carries `renewalPrice` [2][26][27]. Record them, but label them as informational, since Apple says not to use them for revenue recognition [26].
- **Trial detection.** It uses `strtobool(txn["in_trial_period"])`. `distutils` was removed in Python 3.12, so this import will fail on current Python. Use `offerType` = 1 with `offerDiscountType` = `FREE_TRIAL` instead [28][29]. The old code also ignores intro pay-as-you-go/pay-up-front offers, promotional offers, offer codes, and win-back offers; `offerType` and `offerIdentifier` cover all of them [28].
- **Cancellation vs refund.** It treats `auto_renew_status == "1"` as "canceled", which is **inverted**: `1` means auto-renew is on [34]. It also reads `txn["cancellation_date"]` directly, which raises `KeyError` when the field is absent. Use `autoRenewStatus` from renewal info for `renewal.disabled` / `renewal.enabled`, and `revocationDate` / `revocationReason` / `revocationType` on the transaction for `refund.issued` / `revoked` [26][31][32][34].
- **Missing states.** The code has no concept of billing retry, grace period, expiration reason, pending downgrade, price-increase consent, Family Sharing, or upgrades. Map these from `status` [24], `isInBillingRetryPeriod`, `gracePeriodExpiresDate`, `expirationIntent`, `autoRenewProductId`, `priceIncreaseStatus` [27], `inAppOwnershipType` [33], and `isUpgraded` [26].
- **Dates.** The old code stores the receipt's `purchase_date` / `expires_date` strings. JWS fields are UNIX **milliseconds** (`purchaseDate`, `expiresDate`, `originalPurchaseDate`, `signedDate`) [26].
- **Customer linking.** It relies on an `external_user_id` supplied alongside the receipt. Prefer `appAccountToken` from the transaction or renewal info when the app sets it [36].
- **Non-subscription products.** `latest_receipt_info` mixes product types. With the new APIs, filter on `type` = `Auto-Renewable Subscription` [62] or on the `productType` query filter [18], and ignore `ONE_TIME_CHARGE` notifications [5].

## References

1. App Store Server API — https://developer.apple.com/documentation/appstoreserverapi (accessed 2026-09-28)
2. App Store Server API changelog — https://developer.apple.com/documentation/appstoreserverapi/app-store-server-api-changelog (accessed 2026-09-28)
3. App Store Server Notifications — https://developer.apple.com/documentation/appstoreservernotifications (accessed 2026-09-28)
4. App Store Server Notifications changelog — https://developer.apple.com/documentation/appstoreservernotifications/app-store-server-notifications-changelog (accessed 2026-09-28)
5. notificationType — https://developer.apple.com/documentation/appstoreservernotifications/notificationtype (accessed 2026-09-28)
6. subtype — https://developer.apple.com/documentation/appstoreservernotifications/subtype (accessed 2026-09-28)
7. responseBodyV2DecodedPayload — https://developer.apple.com/documentation/appstoreservernotifications/responsebodyv2decodedpayload (accessed 2026-09-28)
8. data (notification payload) — https://developer.apple.com/documentation/appstoreservernotifications/data (accessed 2026-09-28)
9. Enabling App Store Server Notifications — https://developer.apple.com/documentation/appstoreservernotifications/enabling-app-store-server-notifications (accessed 2026-09-28)
10. Responding to App Store Server Notifications — https://developer.apple.com/documentation/appstoreservernotifications/responding-to-app-store-server-notifications (accessed 2026-09-28)
11. Enter server URLs for App Store Server Notifications (App Store Connect Help) — https://developer.apple.com/help/app-store-connect/configure-in-app-purchase-settings/enter-server-urls-for-app-store-server-notifications (accessed 2026-09-28)
12. verifyReceipt — https://developer.apple.com/documentation/appstorereceipts/verify-receipt (accessed 2026-09-28; `Deprecation` response header observed live the same day)
13. App Store Receipts — https://developer.apple.com/documentation/appstorereceipts (accessed 2026-09-28)
14. Creating API keys to authorize API requests — https://developer.apple.com/documentation/appstoreserverapi/creating-api-keys-to-authorize-api-requests (accessed 2026-09-28)
15. Generating JSON Web Tokens for API requests — https://developer.apple.com/documentation/appstoreserverapi/generating-json-web-tokens-for-api-requests (accessed 2026-09-28)
16. Simplifying your implementation by using the App Store Server Library — https://developer.apple.com/documentation/appstoreserverapi/simplifying-your-implementation-by-using-the-app-store-server-library (accessed 2026-09-28)
17. Get All Subscription Statuses — https://developer.apple.com/documentation/appstoreserverapi/get-all-subscription-statuses (accessed 2026-09-28)
18. Get Transaction History — https://developer.apple.com/documentation/appstoreserverapi/get-transaction-history (accessed 2026-09-28)
19. Get Transaction Info — https://developer.apple.com/documentation/appstoreserverapi/get-transaction-info (accessed 2026-09-28)
20. Get Notification History — https://developer.apple.com/documentation/appstoreserverapi/get-notification-history (accessed 2026-09-28)
21. Request a Test Notification — https://developer.apple.com/documentation/appstoreserverapi/request-a-test-notification (accessed 2026-09-28)
22. Get Refund History — https://developer.apple.com/documentation/appstoreserverapi/get-refund-history (accessed 2026-09-28)
23. Identifying rate limits — https://developer.apple.com/documentation/appstoreserverapi/identifying-rate-limits (accessed 2026-09-28)
24. status — https://developer.apple.com/documentation/appstoreserverapi/status (accessed 2026-09-28)
25. expirationIntent — https://developer.apple.com/documentation/appstoreserverapi/expirationintent (accessed 2026-09-28)
26. JWSTransactionDecodedPayload — https://developer.apple.com/documentation/appstoreserverapi/jwstransactiondecodedpayload (accessed 2026-09-28)
27. JWSRenewalInfoDecodedPayload — https://developer.apple.com/documentation/appstoreserverapi/jwsrenewalinfodecodedpayload (accessed 2026-09-28)
28. offerType — https://developer.apple.com/documentation/appstoreserverapi/offertype (accessed 2026-09-28)
29. offerDiscountType — https://developer.apple.com/documentation/appstoreserverapi/offerdiscounttype (accessed 2026-09-28)
30. transactionReason — https://developer.apple.com/documentation/appstoreserverapi/transactionreason (accessed 2026-09-28)
31. revocationReason — https://developer.apple.com/documentation/appstoreserverapi/revocationreason (accessed 2026-09-28)
32. revocationType — https://developer.apple.com/documentation/appstoreserverapi/revocationtype (accessed 2026-09-28)
33. inAppOwnershipType — https://developer.apple.com/documentation/appstoreserverapi/inappownershiptype (accessed 2026-09-28)
34. autoRenewStatus — https://developer.apple.com/documentation/appstoreserverapi/autorenewstatus (accessed 2026-09-28)
35. priceIncreaseStatus — https://developer.apple.com/documentation/appstoreserverapi/priceincreasestatus (accessed 2026-09-28)
36. appAccountToken — https://developer.apple.com/documentation/appstoreserverapi/appaccounttoken (accessed 2026-09-28)
37. originalTransactionId — https://developer.apple.com/documentation/appstoreserverapi/originaltransactionid (accessed 2026-09-28)
38. Reducing Involuntary Subscriber Churn — https://developer.apple.com/documentation/storekit/reducing-involuntary-subscriber-churn (accessed 2026-09-28)
39. Enable Billing Grace Period for auto-renewable subscriptions (App Store Connect Help) — https://developer.apple.com/help/app-store-connect/manage-subscriptions/enable-billing-grace-period-for-auto-renewable-subscriptions (accessed 2026-09-28)
40. Manage pricing for auto-renewable subscriptions (App Store Connect Help) — https://developer.apple.com/help/app-store-connect/manage-subscriptions/manage-pricing-for-auto-renewable-subscriptions (accessed 2026-09-28)
41. Auto-renewable subscription information (App Store Connect Help) — https://developer.apple.com/help/app-store-connect/reference/in-app-purchases-and-subscriptions/auto-renewable-subscription-information (accessed 2026-09-28)
42. Offer auto-renewable subscriptions (App Store Connect Help) — https://developer.apple.com/help/app-store-connect/manage-subscriptions/offer-auto-renewable-subscriptions (accessed 2026-09-28)
43. Supporting offer codes in your app — https://developer.apple.com/documentation/storekit/supporting-offer-codes-in-your-app (accessed 2026-09-28)
44. Supporting win-back offers in your app — https://developer.apple.com/documentation/storekit/supporting-win-back-offers-in-your-app (accessed 2026-09-28)
45. Supporting Family Sharing in your app — https://developer.apple.com/documentation/storekit/supporting-family-sharing-in-your-app (accessed 2026-09-28)
46. Turn on Family Sharing for In-App Purchases (App Store Connect Help) — https://developer.apple.com/help/app-store-connect/configure-in-app-purchase-settings/turn-on-family-sharing-for-in-app-purchases (accessed 2026-09-28)
47. Testing Ask to Buy in Xcode — https://developer.apple.com/documentation/storekit/testing-ask-to-buy-in-xcode (accessed 2026-09-28)
48. Manage Sandbox Apple Account settings (App Store Connect Help) — https://developer.apple.com/help/app-store-connect/test-in-app-purchases/manage-sandbox-apple-account-settings (accessed 2026-09-28)
49. Send Consumption Information — https://developer.apple.com/documentation/appstoreserverapi/send-consumption-information (accessed 2026-09-28)
50. Retention Messaging API — https://developer.apple.com/documentation/retentionmessaging (accessed 2026-09-28)
51. Retention Messaging API changelog — https://developer.apple.com/documentation/retentionmessaging/retention-messaging-changelog (accessed 2026-09-28)
52. Advanced Commerce API — https://developer.apple.com/documentation/advancedcommerceapi (accessed 2026-09-28)
53. Advanced Commerce API changelog — https://developer.apple.com/documentation/advancedcommerceapi/changelog (accessed 2026-09-28)
54. Supporting monthly subscriptions with a 12-month commitment — https://developer.apple.com/documentation/storekit/supporting-monthly-subscriptions-with-a-12-month-commitment (accessed 2026-09-28)
55. Original API for In-App Purchase — https://developer.apple.com/documentation/storekit/original-api-for-in-app-purchase (accessed 2026-09-28)
56. SKPaymentQueue — https://developer.apple.com/documentation/storekit/skpaymentqueue (accessed 2026-09-28)
57. App information (App Store Connect Help) — https://developer.apple.com/help/app-store-connect/reference/app-information/app-information (accessed 2026-09-28)
58. App Store Server Library for Python (Apple, GitHub; README and releases) — https://github.com/apple/app-store-server-library-python (accessed 2026-09-28)
59. Apple In-App Purchase (StoreKit) — https://developer.apple.com/documentation/storekit/in-app-purchase (accessed 2026-09-28)
60. billingPlanType — https://developer.apple.com/documentation/appstoreserverapi/billingplantype (accessed 2026-09-28)
61. Auto-renewable subscription price increase thresholds (App Store Connect Help) — https://developer.apple.com/help/app-store-connect/reference/in-app-purchases-and-subscriptions/auto-renewable-subscription-price-increase-thresholds (accessed 2026-09-28)
62. type (In-App Purchase product type) — https://developer.apple.com/documentation/appstoreserverapi/type (accessed 2026-09-28)
63. HistoryResponse — https://developer.apple.com/documentation/appstoreserverapi/historyresponse (accessed 2026-09-28)
