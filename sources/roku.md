# Roku Pay

> Last reviewed: 2026-09-28 · Current versions: Roku Pay web services have no published version number (base path `transaction-service.svc`). Page `updatedAt` stamps: web services reference 2026-09-22 [1], setup 2026-09-22 [2], push notifications reference 2026-09-22 [3], JWT push guide 2026-06-02 [4], Enhanced Subscription Recovery 2026-09-22 [5], Basic Subscription Recovery 2026-07-20 [6], on-device upgrade/downgrade 2026-09-22 [7], product catalog (Catalog 2.0) 2026-09-11 [8], Roku Pay requirements 2026-05-11 [11]. Latest Roku OS release notes: Roku OS 16.0 [16]. Many pages share the same 2026-09-22T21:46:36Z stamp, which looks like a site-wide republish rather than a content change to each page.

## Overview

Roku Pay is Roku's billing system for subscriptions and one-time purchases in Roku apps (Roku still calls these "channels" in field names such as `channelId`). The publisher defines **products** and their **purchase options** (billing frequency, price tier, free trial or introductory price) in the Roku Developer Dashboard [8]. Inside the app, the SceneGraph `ChannelStore` node (or the older BrightScript `roChannelStore`) shows Roku's own screens. `getUserData` shows the Request for Information screen, `getCatalog` lists products, and `doOrder` shows the order confirmation screen and completes the purchase. `getAllPurchases` returns the customer's existing purchases at app launch [10][13]. Roku charges the customer's payment method on file and handles renewals, dunning, tax, currency conversion and payouts [14]. Every app that sells subscriptions or one-time purchases must use Roku Pay to pass certification [11].

On the server side, the publisher's backend uses two channels. The **Roku Pay web services** are a small REST API: validate a transaction, validate a refund, cancel, refund, and issue a service credit [1]. **Push notifications** are server-to-server messages, JWT-signed, that Roku sends to one configured URL whenever a sale, renewal, cancellation, refund, credit, grace/on-hold change, upgrade/downgrade or chargeback happens [2][3][4]. Roku asks publishers to use both: push for near-real-time updates, and a nightly `validate-transaction` sync as a backstop [1][5]. For Subsist, the push stream is the primary event source and `validate-transaction` is the re-sync and verification path.

## APIs and versions

All web-service endpoints share the base URL `https://apipub.roku.com/listen/transaction-service.svc`. They accept JSON or XML, selected with the `accept` header (`application/json` or `application/xml`). GET requests carry the API key and the ID in the URL path. POST requests carry the API key in the body as `partnerAPIKey` [1]. The docs say HTTP or HTTPS may be used [1]. Subsist should always use HTTPS.

| API | Endpoint | What Subsist uses it for | Status |
| :-- | :-- | :-- | :-- |
| validate-transaction | `GET /validate-transaction/{partnerAPIKey}/{transactionid}` | Fetch current state of a subscription or purchase (`isEntitled`, `expirationDate`, `cancelled`, `purchaseStatus`, `purchaseType`, amounts). Used for backfill, nightly re-sync, and to confirm push events. | Current [1] |
| validate-refund | `GET /validate-refund/{partnerAPIKey}/{refundId}` | Confirm a refund Subsist (or the customer's support tooling) issued; returns negative `amount`/`total`. | Current [1] |
| cancel-subscription | `POST /cancel-subscription` (body: `cancellationDate`, `dontNotifyUser`, `partnerAPIKey`, `partnerReferenceId`, `transactionId`) | Not needed for ingestion. Optional support action. Roku also tells publishers to call it once the nightly sync sees `isEntitled` = false after recovery fails [1]. | Current [1] |
| refund-subscription | `POST /refund-subscription` (body: `amount`, `comments`, `partnerAPIKey`, `partnerReferenceId`, `transactionId`) → `RefundId` | Optional support action. The resulting `Refund` push is what Subsist records. | Current [1] |
| issue-service-credit | `POST /issue-service-credit` (body: `partnerAPIKey`, `amount`, `channelId`, `comments`, `partnerReferenceId`, `productId`, `rokuCustomerId`) → `ReferenceId` | Optional support action. The resulting `Credit` push is recorded. | Current [1] |
| update-bill-cycle | `POST /update-bill-cycle` (body: `rokuAPIKey`, `newBillCycleDate`, `transactionId`) | Do not depend on it. | **Not in the current reference.** It appears only in an older, unofficial mirror of Roku's docs [19]. The current "Implementing Roku Pay" page still says the web services can update "the customer's billing cycle" [13], but the current API reference lists only the five endpoints above [1]. Treat it as legacy and unverified. |
| Push notifications | Publisher HTTPS endpoint set in the Developer Dashboard | Primary lifecycle event feed. | Current. JWT/JWS signing is mandatory for all developer accounts since 2024-02-01 [2][4]. |
| ChannelStore `getAllPurchases` / v2 `GetPurchases` | On-device only (BrightScript) | Not callable by Subsist. Documented here because the customer's app sees states (for example `ActivePaused`) that the server APIs do not expose [9][10]. | Current |
| ChannelStore `GetRokuCustomerId` | On-device only | Lets the app learn `rokuCustomerId` before any purchase, so app-side user IDs can be joined to push notifications [10][16]. | New in Roku OS 16.0 [16] |

**Rate limit:** 20 requests per second per API key. Excess requests get HTTP `429` with an empty body. Roku recommends exponential backoff starting at 1 second and doubling up to 60 seconds, and spreading nightly reconciliation over a window sized to the subscriber count [1].

**Transaction IDs** are ASCII strings of variable length, up to 1024 bytes [1][3]. Examples include 32-character hex strings and hyphenated UUIDs [3].

## Credentials required

All of these are found in the Roku Developer Dashboard (`https://developer.roku.com/developer`). The Roku Pay settings live on the **Roku Pay Web Services** page in the left sidebar (`https://developer.roku.com/api/settings`), which has tabs for the API key, push notifications, and allowed IP ranges [2][4].

| Item | What it is | Where the customer gets it |
| :-- | :-- | :-- |
| **Roku Pay API key** (`partnerAPIKey`) | The single secret for all web-service calls. Roku also describes it as enabling push notifications [2]. Every call must use the key of the developer account that owns the app behind the transaction or refund ID [1]. | Roku Pay Web Services → **Roku Pay API Key** tab. At most two keys exist at once, one active and one expired or expiring. **Invalidate** creates a new key and expires the old one immediately or after 1–30 days. An expired key can be reactivated for 1–365 days [2]. Subsist should accept a second key during rotation. |
| **Allowed IP address range** | Optional IPv4 allow-list. If the customer has set one, calls from any other IP are rejected [2]. | Roku Pay Web Services → **Allowed IP address range** tab. Customers who use this must add Subsist's egress IPs, so Subsist needs stable egress IPs. |
| **Push notification URL** | The HTTPS endpoint Roku posts notifications to. There is also a separate **Test push notification URL** with an end time, a **Send test message** button, and **Replay notifications** [2]. | Roku Pay Web Services → **Push notifications** tab. The Instant Signup guide also mentions a **Stop sending billing notifications** checkbox that must stay cleared [12]. The docs describe one production URL field and do not describe per-app URLs, so it appears to be one URL per developer account (unverified). A customer who already consumes pushes would need Subsist to forward them, or would have to forward them to Subsist. |
| **JWT public keys** | Not a secret. These are Roku's RS256 signing keys. | Production: `https://assets.cs.roku.com/keys/partner-jwks.json`. Test endpoint: `https://assets.cs.roku.com/keys/partner-jwks-test.json` [4]. |
| **Channel (app) ID** | `channelId` appears in validate responses and push payloads [1][3]. Subsist uses it to route events to the right customer app. | Shown for each app in the Developer Dashboard app list (exact UI location unverified). It is also present in every push payload. |
| **Product IDs / SKUs** | The purchase option **SKU**, which "is used in the Roku Pay APIs and reporting" and cannot be changed after publishing [8]. It appears as `productId` in validate responses and `productCode` in push payloads. Under Catalog 2.0, `productId`/`productName` refer to the **purchase option**, not the product, so a monthly and an annual option of one product have different IDs [9]. | Developer Dashboard → product catalog → **Purchase options** tab [8]. Subsist should import the SKU → plan mapping (billing frequency, price tier, trial/intro offers) from the customer. |

## Server notifications

### Setup and transport

- **Configure:** First test on the test URL, which must be HTTPS and has an end time. Then set the production URL [2].
- **Signing:** Since 2024-02-01 every account receives JWT/JWS-signed messages and cannot go back to unsigned ones [2]. The request has `Content-Type: text/plain`. The body is a JWS compact serialization, `header.payload.signature` [4].
  - JOSE header: `"typ": "JWT"`, `"alg": "RS256"`, `"kid"` (for example `ROKU-PARTNER-SERVICE-2021-04-29`). Pick the matching key from the JWKS [4].
  - Claims: `iss` = `Roku, Inc. urn:roku:apps:partner-service.roku.com`; `exp` = 24 hours after generation; `nbf` = one hour before generation; `x-Roku-message` = base64url of the UTF-8 JSON notification; `x-Roku-message-encoding` = `base64-utf8`; `x-Roku-message-key` = unique key "used to de-duplicate messages"; `x-Roku-message-type` = `roku.rpay.push` [4].
  - Roku says to reject messages whose key URL is not the fixed JWKS URL above [4].
- **Acknowledge:** The JWT guide says to reply `200 OK` [4]. The push reference, which still describes the older unsigned flow, says to reply with the `responseKey` value as a text body, and that Roku checks its size [3]. Replying `200 OK` with the `responseKey` as a `text/plain` body satisfies both descriptions.
- **Delivery rules:** Roku does not follow redirects, and a redirect counts as a failure. Requests time out after 10 seconds [3]. If one message keeps failing for 36 hours, Roku stops sending it. If the endpoint fails to acknowledge 100 notifications within 10 days, it is put on a deny list [2]. Keep the handler fast: verify, store, return 200, and process asynchronously.
- **Replay:** The customer can replay any 14-day window within the past 90 days from the dashboard. Replayed messages must be processed in timestamp order. Roku gives the example that `OnHoldInitiated` followed by `OnHoldRecovered` processed out of order leaves the user wrongly locked out [2]. Order by `eventDate` and dedupe on `x-Roku-message-key` and `transactionId`.
- **Reference receiver:** `https://github.com/rokudev/notification-receiver-sample` [4].

### Payload fields

Decoded `x-Roku-message` JSON uses these fields, copied from the examples in [3][4][12]:

`customerId`, `transactionType`, `transactionId`, `channelId`, `channelName`, `productCode`, `productName`, `price`, `tax`, `total`, `currency`, `isFreeTrial`, `expirationDate`, `originalTransactionId`, `originalPurchaseDate`, `eventDate`, `comments`, `responseKey`, `purchaseChannel`, `purchaseContext`, `partnerReferenceId`, `creditsApplied`.

Notes:
- Dates are ISO 8601 UTC with `Z`. The examples show 0, 7 and 9 fractional-second digits, so parse tolerantly [3][4]. The validate-transaction JSON uses a different date format (see below).
- `creditsApplied` has been present only when a service credit was applied, since 2020-03-23 [3].
- Not every field appears on every type. Cancellation and grace/on-hold messages have no `price`/`total`, and a first `Sale` may lack `originalPurchaseDate` [3].
- `purchaseChannel`/`purchaseContext` are `DEVICE`/`IAP` for on-device purchases, `WEB`/`ISU` for Instant Signup on the web, and `DEVICE`/`ISU` for on-device Instant Signup offers. The push examples use upper case. The validate-transaction docs show lower case (`web`, `isu`) [1][3][12].
- **PII:** when the customer consents during Instant Signup, the `Sale` payload also carries `email`, `zip`, `gender`, `firstName`, `lastName`, `birthMonth`, `birthYear`, `federationToken` and `pucId` [12]. Subsist should drop or segregate these fields unless the customer explicitly wants them.
- The push field is `customerId`, but validate-transaction calls it `rokuCustomerId`. Likewise the push field `productCode` corresponds to `productId` in validate-transaction.

### Notification types → Subsist lifecycle events

| `transactionType` | Meaning (per Roku) | Subsist lifecycle event |
| :-- | :-- | :-- |
| `Sale`: new order, `isFreeTrial` = true | A free trial started, on-device or through Instant Signup [3][12]. | `trial.started` (plus `subscription.started` if Subsist models trials as subscriptions) |
| `Sale`: new order, `isFreeTrial` = false | A new paid purchase. `comments` is `"New order processed."` [3]. | `subscription.started`. If an introductory price applied, also `offer.redeemed` (inferred from price vs. catalog price; the payload has no intro-offer flag). |
| `Sale`: renewal | A renewal charge. `comments` is `"Recurring subscription processed"`. This includes the first charge after a free trial [3]. | `payment.renewed`. If the prior known state was a trial, emit `trial.converted` instead of, or in addition to, `payment.renewed`. |
| `GraceInitiated` | Auto-renew payment failed and the subscription entered the 3-day grace period. The customer keeps access. `comments` = `"Subscription is in dunning state"` [3]. | `grace_period.started` (and `payment.retrying`; Roku keeps charging the payment method but sends no per-attempt events) |
| `GraceRecovered` | Payment collected during grace. The billing period is unchanged [3]. | `payment.renewed` |
| `OnHoldInitiated` | Grace ended without payment, so the subscription is on hold and access must be blocked. Sent only to apps on Enhanced Subscription Recovery [3][5]. | `subscription.on_hold` (Roku keeps retrying, which is `payment.retrying` implicitly) |
| `OnHoldRecovered` | Payment collected while on hold. Access is restored and the billing period is re-anchored to the payment date [3][5]. | `payment.renewed`. Record the new billing anchor. Some Subsist consumers may also want `subscription.resumed`; Roku does not call this a resume. |
| `CancellationOfferInitiated` | The customer accepted a cancellation (retention) offer while turning off auto-renew at my.roku.com. The offer's price and terms apply [3][8]. | `offer.redeemed` (the price change takes effect at the next billing cycle [8]) |
| `CancellationOfferEnded` | The cancellation offer's price and terms have elapsed [3]. | `price.changed` (the subscription returns to the regular price; inferred). Roku's "action required" text for this row repeats the `Cancellation` wording, which looks like a copy-paste in the docs. |
| `Cancellation`: `expirationDate` in the future | The customer actively cancelled or turned off auto-renew. Access continues until `expirationDate` [3]. | `renewal.disabled` |
| `Cancellation`: `expirationDate` = today | Active cancellation and today is the last day [3]. | `renewal.disabled`, then `subscription.expired` at `expirationDate` |
| `Cancellation`: `expirationDate` in the past | Passive cancellation because payment could not be recovered after grace/on-hold [3]. Also sent after a free trial ends with a failed payment on Basic recovery [6]. | `subscription.expired` |
| `Refund` | Refund issued by the publisher or Roku. `price`/`tax`/`total` are negative. If the refund was for an unauthorized purchase, Roku also cancels the subscription and sends a separate `Cancellation` [3]. | `refund.issued`. If a `Cancellation` for the same `originalTransactionId` follows, emit `revoked` (inferred pairing, not a documented flag). |
| `Credit` | Service credit issued by the publisher or Roku. No action needed [3]. | — (record as a ledger credit; credits are consumed against later charges and show up as `creditsApplied` [1][3]) |
| `Resubscribe` | A customer who cancelled undid it within the same billing period. Service continues as if they never cancelled. A repurchase after the period ends arrives as a `Sale` instead [3]. | `renewal.enabled` |
| `UpgradeSale` | The upgraded plan was purchased. A prorated credit from the old plan is applied [3][7]. | `plan.changed` (new `transactionId`/`originalTransactionId` starts; `trial.started` too if `isFreeTrial` = true) |
| `UpgradeCancellation` | The original plan was cancelled because of the upgrade [3]. | — (fold into the `plan.changed` above; do not emit `subscription.expired`) |
| `DowngradeSale` | The downgraded plan was purchased. It takes effect at the current plan's `expirationDate`, at $0 until then [3][7]. | `plan.changed` (scheduled; effective at `expirationDate`) |
| `DowngradeCancellation` | The original plan was cancelled because of the downgrade and ends at its expiration date [3]. | — (fold into `plan.changed`) |
| `Chargeback` | The customer disputed the transaction. The amount is deducted from the publisher payout. No action needed [3]. This type also covers SEPA chargebacks and insufficient-funds cases in Germany [3]. | `refund.issued` (reason: chargeback). Roku documents no entitlement change. |
| `ChargebackReversed` | Roku won the dispute and the revenue share is returned [3]. | — (record as a reversal of the chargeback amount) |
| `SecondChargeback` | The bank disputed the reversal and the amount is deducted again [3]. | `refund.issued` (reason: second chargeback) |

Subsist events with no Roku source: `subscription.paused` and `subscription.resumed` have no push type. The on-device v2 `GetPurchases` lists `ActivePaused` and `InactivePaused` states [9], but no server API or notification exposes pause, and no Roku doc describes how a pause happens. `price.changed` for publisher-scheduled price changes also has no notification. It must be inferred from a renewal `Sale` whose `price` differs from the previous one, or from the customer's catalog [8].

## Lifecycle rules and edge cases

### validate-transaction response fields

Documented JSON fields [1][7]: `errorCode`, `errorDetails`, `errorMessage`, `status`, `OriginalTransactionId` (capital O), `amount`, `cancelled`, `cancelledTransactionIds`, `channelId`, `channelName`, `couponCode`, `currency`, `expirationDate`, `isEntitled`, `originalPurchaseDate`, `partnerReferenceId`, `purchaseChannel`, `purchaseContext`, `productId`, `productName`, `purchaseDate`, `purchaseStatus`, `purchaseType`, `quantity`, `rokuCustomerId`, `tax`, `total`, `transactionId`. The validate-refund response also documents `creditsApplied` [1], and the upgrade guide says the upgrade transaction's `creditsApplied` holds the prorated credit [7].

Quirks to handle:
- **Status:** `status` is `0` in JSON and `Success` in XML [1].
- **Dates:** JSON dates use the .NET form `/Date(1581033062000+0000)/`, sometimes JSON-escaped as `\/Date(...)\/`. XML dates are ISO without a zone, for example `2020-02-06T23:51:02` [1][7].
- **Optional fields:** `purchaseType` and `cancelledTransactionIds` are shown only in upgrade/downgrade examples. `purchaseType` is `UPGRADE`, `DOWNGRADE` or `null`. `cancelledTransactionIds` is a JSON **array** in JSON and a single element in XML [1][7].
- **`channelId`** is a number in some examples and a string such as `"000000"` in others [1][7].
- **`purchaseStatus` values** appear as `Active`, `Inactive`, `PendingActive` and `PendingInactive` in JSON. The prose tables spell them `Pending_Active` and `Pending_Inactive`, and one sentence says the value becomes `valid` at activation. The prose also uses snake_case names (`purchase_type`, `purchase_status`, `is_entitled`) for camelCase fields [1][7]. Normalize case, underscores and spelling.
- **Conflicting docs on `Pending_Active`:** the web-services reference says `isEntitled` = false for `Pending_Active` [1], while the upgrade guide's table and its JSON example show `isEntitled` = true [7]. Do not derive entitlement from `purchaseStatus` alone.
- **Refunds:** in validate-refund, `amount` and `total` are negative, `expirationDate` is `null`, and `isEntitled` is false [1].
- **Instant Signup:** `purchaseChannel`/`purchaseContext` are `web`/`isu` for Instant Signup and `device`/`iap` for on-device purchases [1].
- **Catalog 2.0:** `productId` and `productName` refer to the purchase option [9].

### State from validate-transaction

This is Roku's documented truth table for Enhanced Subscription Recovery [5]:

| State | `isEntitled` | `expirationDate` | `cancelled` |
| :-- | :-- | :-- | :-- |
| Current | true | future | false |
| In grace (3 days) | true | current or past | false |
| On hold | false | current or past | false |
| Cancelled (ended) | false | past | true |
| Cancelled, pending end of term | true | future | true |

Apps on Basic recovery have no on-hold row [6].

### Free trials and introductory offers

- **Offer types:** Offers live on the purchase option. The base offer is none, a free trial (in days or months), or an introductory price (in days, months or years, at a lower price tier). Limited-time offers override the base offer while they are active [8]. Retention offers are separate "cancellation offers" [8].
- **One offer per customer:** A customer receives at most one free trial or discount per subscription product, ever [8].
- **Trial flag:** `isFreeTrial` appears on push payloads [3]. validate-transaction has **no documented trial flag**. A $0 `total` does not reliably mean a trial, because upgrade and downgrade transactions also show `total` 0.0000 [7].
- **Trial end with a failed payment:** on Enhanced recovery, `isEntitled` becomes false and the subscription goes **on hold**. On Basic recovery it is cancelled immediately, with no grace period [1][5][6].
- **Trial conversion:** arrives as a renewal `Sale` ("renewal of a free trial") [3].
- **Instant Signup:** free trials started during device activation arrive as `Sale` with `purchaseContext` `ISU` [12].

### Upgrades and downgrades

- **Product groups required:** Plans must be in the same product exclusivity group. The app sends `doOrder` with `order.action` = `Upgrade` or `Downgrade` (case-sensitive), then validates the new `purchaseid` [7][8].
- **Upgrade:** The old plan is cancelled now and the new plan is bought with a prorated service credit. The old transaction shows `cancelled` = true with an unchanged `expirationDate`. Without a trial on the new plan, the old one is immediately `Inactive`. With a trial, it is `PendingInactive`. If the customer then cancels the upgrade, the old plan is reinstated but does not renew, and it becomes `Inactive` after the upgraded plan's first successful renewal [1][7].
- **Downgrade:** The current plan is marked cancelled at its `expirationDate`. A new transaction appears at $0 with the same `expirationDate` and `purchaseStatus` `PendingActive`. At expiry the downgraded plan is charged under a new transaction ID. No credit is issued [7].
- **Linking:** The new transaction's `cancelledTransactionIds` holds the replaced transaction ID [1][7]. Subsist should link the two into one subscription lineage, because `OriginalTransactionId` is new for the upgraded or downgraded plan [7].
- **Add-ons:** An add-on survives a base-plan upgrade or downgrade only if the new order includes it. Cancelling every prerequisite base plan cancels the add-on [9].

### Cancellations and resubscribe

- **Active cancellation** keeps access until `expirationDate`. **Passive cancellation** has a past `expirationDate` [3].
- **Resubscribe:** A cancel that is undone within the same period sends `Resubscribe`. A purchase after the period ends is a new `Sale` [3].
- **Publisher-initiated cancel:** `cancel-subscription` takes `cancellationDate` and `dontNotifyUser` [1].
- **Archiving:** Archiving an ended purchase option cancels all its subscriptions at the end of their billing cycle [8]. This is a bulk source of `Cancellation` events.

### Refunds

- **Amount rules:** A refund `amount` must be greater than $0, at most the pre-tax price, and **tax-exclusive**. Roku adds the tax itself; for example, refunding $5.00 of a $10 + 10% tax charge refunds $5.50 [1].
- **Cap:** Partial refunds on one transaction cannot sum to more than the original amount [1].
- **Unauthorized purchases:** Roku also cancels the subscription and sends a separate `Cancellation` [3].

### Service credits

- **How they work:** A credit acts as the customer's payment method until its balance reaches $0.00, then the card on file is charged [1].
- **Scope:** A credit can apply to an app (`channelId`) or to one product (`channelId` + `productId`) [1].
- **Where they show up:** Applied credits appear as `creditsApplied` on later `Sale`/`Resubscribe` pushes [3] and as `service_credits` in the Transaction Report [15]. Revenue should be computed net of credits.

### Bill-cycle changes

- **No server-side notification:** Roku documents no notification for a bill-cycle change.
- **Implicit re-anchor:** The billing anchor moves implicitly on `OnHoldRecovered`, which re-anchors to the payment date [3][5]. It also moves on upgrades, where `expirationDate` is set from the new plan's term [7].
- **Legacy API:** `update-bill-cycle` is not in the current API reference (see above) [1][19].

### Failed payments and retry

- **Grace:** 3 days with access, and Roku emails the customer daily [3][5].
- **On hold:** Enhanced recovery only. Up to 57 more days without access, with home-screen, app-launch, in-app and email prompts. That makes a 60-day total cycle before cancellation [5].
- **Recovery:** A payment during grace keeps the billing period. A payment while on hold re-anchors it to the payment date [5].
- **Enhanced recovery is mandatory:** Every subscription app has needed Enhanced Subscription Recovery to pass certification since **2024-10-01**, and Basic-only apps must migrate [6][11].
- **Retry cadence:** Roku does not publish the number or timing of retry charges.
- **Status check:** `DoRecovery` is an on-device ChannelStore command. It shows the renewal dialog and returns `recoveryStatus` 1, 2 or 3 [5].

### Entitlement checks

- **Account-wide:** Roku requires entitlements to be account-wide across all devices on the purchasing Roku account (RP 4.3) [11].
- **At launch:** The app calls `getAllPurchases` and passes the `purchaseId` to the backend, which calls `validate-transaction` [13].
- **Pre-Catalog 2.0 devices:** `getAllPurchases` returns `status` (`Valid`/`Invalid`) and `inDunning`. `inDunning` = true with `Valid` means grace; `inDunning` = true with `Invalid` means on hold [5][10].
- **Catalog 2.0 v2 `GetPurchases`:** returns `billingPlans[].state` with the values `ActivePaid`, `ActiveFreeTrial`, `ActiveCanceled`, `ActiveInGracePeriod`, `ActivePaused`, `InactiveWaitingActivation`, `InactivePaused`, `InactiveOnHold` and `InactiveExpired`, plus `renewalDate`, `subscriptionId` and `phases[]` with offer types `FreeTrial`/`ReducedPrice`/`RegularPrice` [9]. None of this richer state is available server-side.

### Price changes

- **Scheduling:** Price changes are scheduled per purchase option, for new or existing subscribers. For new subscribers the earliest effective time is midnight the next day. For existing subscribers the dashboard requires the date to be 15 days out [8].
- **Notice period:** Roku's own pages disagree. RP 3.3 says at least 15 days [11]. The product-catalog page says both "a 30-day notice" and "at least 7 days prior but no more than 30 days" [8].
- **No notification:** No push is sent when a price change takes effect.

### Testing data

- **Test users:** Test users on the billing-test app make real ChannelStore calls without being charged [17][18].
- **Forcing states:** A **subscription-recovery** test API can force active, grace, on-hold, passively cancelled and recovered states [5]. Its endpoint details are on the Testing page and were not verified here.
- **Voiding:** Test transactions can be voided from **Manage Test Users** in the dashboard.
- **Filtering:** Subsist should expect test traffic from customers and let them filter it. A $0 "Purchase" in the Transaction Report is either a free trial or a test transaction [15].

## Checking current status (re-sync)

- **`validate-transaction`** is the only server-side status endpoint [1]. It takes one transaction ID per call. There is no "list subscriptions for customer" or "list transactions since" API.
  - Call it with the subscription's latest known transaction ID or its original transaction ID. The recovery guidance says to call it "with the `transactionId` of the subscription" and read the updated `expirationDate` [1].
  - Which ID returns the freshest renewal state is not spelled out (unverified). Store both and prefer the most recent.
- **Nightly sync, as Roku prescribes:** each night, select subscriptions whose `expirationDate` is today or earlier and call `validate-transaction` for each. Spread the calls over about 6 hours [1]. Then:
  - `isEntitled` true and `expirationDate` moved forward → renewed.
  - `isEntitled` true and `expirationDate` unchanged → still in grace.
  - `isEntitled` false and `cancelled` false → on hold (Enhanced) [5].
  - `isEntitled` false and `cancelled` true → cancelled. Roku says to stop polling it and optionally call `cancel-subscription` [1].
- **Throttling:** Stay under 20 requests per second per customer API key. On `429`, back off from 1 s, doubling up to 60 s [1]. For large customers, focus on subscriptions near expiry or in dunning [1].
- **`validate-refund`** confirms a specific `RefundId` [1].
- **Missed pushes:** The customer can use **Replay notifications** (any 14-day window in the last 90 days) to backfill [2].
- **Deeper history:** The dashboard **Transaction Report** has history back to 2018-01-01 with `transaction_type`, `original_transaction_id`, `expiration_date`, `service_credits` and `net_amount`. It is export-only from the dashboard, with no API documented [15]. It is the practical source for the initial backfill of transaction IDs, which Subsist then enriches with `validate-transaction`.

## Migration notes for subsist-app

> **Status (2026-09-28):** implemented in `subsist-app` (not yet deployed). `roku/client.py` wraps the Roku Pay web services with typed errors, a per-key rate limit and the documented 429 backoff; `roku/sync.py` validates transactions, applies push payloads (ignoring out-of-order replays) and links upgrades/downgrades via `originalTransactionId`; `roku/views.py` verifies signed (RS256 JWS) pushes at `/roku/notifications/<app id>/`, deduplicates, strips Instant Signup PII and answers with 200 plus the `responseKey`; `manage.py roku_sync` implements the nightly Enhanced Subscription Recovery sync. Still open: encrypting the API keys, a product catalog for trial/intro-offer detection outside pushes, and optional `cancel-subscription` calls for ended subscriptions.

What `subsist-app/roku/models.py` (2022) does today, and what should change:

- **Pull only.** The code only calls `validate-transaction` (`RokuRequester.fetch`) with the correct base URL and GET path. There is no push-notification ingestion at all, so it cannot see grace, on-hold, refunds, credits, resubscribes, upgrade/downgrade pairs or chargebacks. Add a push endpoint that:
  - verifies the JWS against `https://assets.cs.roku.com/keys/partner-jwks.json` by `kid` (RS256; check `iss`, `exp` and `nbf`);
  - decodes `x-Roku-message`;
  - dedupes on `x-Roku-message-key`;
  - returns `200` with the `responseKey` body within 10 s [3][4].
- **Renewal status is inverted.** `renewal_status = "Active" if self.cancelled else "Canceled"` is backwards; `cancelled` = true means auto-renew is off. Map it from `cancelled` and `isEntitled` using the state table above [5].
- **Free-trial detection is wrong.** `payment_status = "Free Trial" if str(self.total) == "0.0000"` also flags upgrades and downgrades (which show `total` 0.0000 [7]) and credit-covered charges as trials. The string compare is also fragile against `Decimal` formatting. Use the push `isFreeTrial` flag, and for validate-only data fall back to catalog offer data.
- **Hard `response["..."]` indexing will raise `KeyError`.** `purchaseType` and `cancelledTransactionIds` appear only on upgrade/downgrade responses [1][7], and `purchaseChannel`/`purchaseContext` are newer fields. Use `.get()`. Store `purchaseChannel` and `purchaseContext`, which `detect_drift` currently only prints. Also store `creditsApplied`.
- **`cancelledTransactionIds` is an array** [1]. It is stored in a `TextField` as a Python list repr. Store it as JSON and use it to link upgrade/downgrade lineage.
- **Date parsing is brittle.** `ROKU_DATE_FORMAT` requires exactly 13 digits and a `+` offset, and returns `None` otherwise. `None` then goes into non-nullable `expiration_date`, `purchase_date` and `original_purchase_date`, and validate-refund returns `expirationDate: null` [1]. Accept any digit count and `±` offsets, and make `expiration_date` nullable. Push payloads use ISO 8601 with up to 9 fractional digits [3][4] and need a separate parser.
- **`purchase_status`** is stored raw. Normalize `PendingActive` / `Pending_Active` / `pending_active` and the others [1][7], and never use it alone for entitlement (see the conflicting docs above).
- **`channel_id` is an `IntegerField`,** but Roku returns strings like `"000000"` in some responses [7]. Store it as text.
- **No handling of `status` or `errorCode`.** A non-zero `status` or a populated `errorCode` is currently saved as a normal receipt, and a non-2xx response silently returns `None`. Treat API errors explicitly and handle `429` with backoff (1 s doubling to 60 s) and a 20-rps per-key limiter [1].
- **`billing_cycle_start = purchase_date` / `billing_cycle_end = expiration_date`** is acceptable for the current period only. After `OnHoldRecovered` the cycle re-anchors to the payment date [5], and a downgrade's `PendingActive` row has a future start. Model periods from push events.
- **`subscription_id = original_transaction_id`** is right within one plan, but upgrades and downgrades create a new `OriginalTransactionId` [7]. Link them through `cancelledTransactionIds` or the `UpgradeCancellation`/`DowngradeCancellation` pair.
- **`brand="brand"` is hard-coded.** Derive the brand from the customer's `channelId` → app mapping.
- **Model-field naming:** `cancelled_transactionids` should be renamed for consistency with Roku's `cancelledTransactionIds` when the model is migrated.
- **Missing endpoints:** add `validate-refund`, and optionally `cancel-subscription`, `refund-subscription` and `issue-service-credit`. Do **not** build on `update-bill-cycle` [1][19].
- **Credentials:**
  - Support two API keys per customer for rotation [2].
  - Keep egress IPs stable for customers who use the IP allow-list [2].
  - Redact the API key from logs, because GET requests put it in the URL path [1].
- **Catalog 2.0:** `productId` is now the purchase-option SKU [9]. Import the customer's SKU → product/plan mapping instead of treating `productId` as a plan.
- **Recovery mode:** Assume Enhanced Subscription Recovery, which has been mandatory since 2024-10-01 [6][11], and add the on-hold state and events.

## References

1. Roku Pay web services reference — https://developer.roku.com/dev/docs/roku-web-service (accessed 2026-09-28)
2. Setting up Roku Pay web services — https://developer.roku.com/dev/docs/setting-up-web-services (accessed 2026-09-28)
3. Roku Pay push notifications reference — https://developer.roku.com/dev/docs/push-notifications (accessed 2026-09-28)
4. Receiving secured Roku Pay push notifications — https://developer.roku.com/dev/docs/push-notifications-jwt (accessed 2026-09-28)
5. Enhanced Subscription Recovery — https://developer.roku.com/dev/docs/subscription-on-hold (accessed 2026-09-28)
6. Basic Subscription Recovery — https://developer.roku.com/dev/docs/basic-recovery (accessed 2026-09-28)
7. On-device upgrade and downgrade — https://developer.roku.com/dev/docs/on-device-upgrade-downgrade (accessed 2026-09-28)
8. Creating the product catalog — https://developer.roku.com/dev/docs/product-catalog (accessed 2026-09-28)
9. Catalog 2.0 API integration guide — https://developer.roku.com/dev/docs/add-ons-integration (accessed 2026-09-28)
10. ChannelStore — https://developer.roku.com/dev/docs/channelstore (accessed 2026-09-28)
11. Roku Pay integration requirements — https://developer.roku.com/dev/docs/roku-pay-requirements (accessed 2026-09-28)
12. Instant Signup — https://developer.roku.com/dev/docs/instant-signup (accessed 2026-09-28)
13. Implementing Roku Pay — https://developer.roku.com/dev/docs/implementation (accessed 2026-09-28)
14. Subscriptions and one-time purchases — https://developer.roku.com/dev/docs/billing (accessed 2026-09-28)
15. Transaction Report — https://developer.roku.com/dev/docs/transaction-report (accessed 2026-09-28)
16. Roku OS developer release notes — https://developer.roku.com/dev/docs/release-notes (accessed 2026-09-28)
17. Testing Roku Pay — https://developer.roku.com/dev/docs/testing (accessed 2026-09-28)
18. Enabling billing testing — https://developer.roku.com/dev/docs/billing-testing (accessed 2026-09-28)
19. Unofficial mirror of older Roku web-services docs (legacy `update-bill-cycle`) — https://github.com/alaba19/docs-1/blob/master/develop/guides/roku-web-services.md (accessed 2026-09-28)
