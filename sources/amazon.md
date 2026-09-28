# Amazon Appstore

> Last reviewed: 2026-09-28 · Current versions: Appstore SDK 3.0.9 (released 2026-05-20) [4]; RVS `verifyReceiptId` operation version 1.0 (RVS docs last updated 2026-05-20) [7]; Acknowledge Receipt API (`acknowledgeReceipt`) version 1.0 [22]; Real-Time Notifications has no published payload version (RTN docs last updated 2025-07-22) [12] · Platform status: Amazon Appstore for non-Amazon Android phones and tablets was discontinued on 2025-08-20; the Appstore on Fire TV, Fire tablets and Fire TV built-in products continues and is supported [1][2].

## Overview

Amazon In-App Purchasing (IAP) sells three kinds of items: consumables, entitlements and subscriptions [23]. A subscription is modeled as a non-buyable **parent SKU** that represents the product, plus one or more **term SKUs** (child SKUs) that each represent one billing period and price. The customer buys a term SKU, but the purchase response reports the parent SKU; this structure prevents a customer from buying the same product twice [15]. Term periods are Weekly, BiWeekly, Monthly, BiMonthly, Quarterly, SemiAnnually and Annually, and each term can carry an optional free trial [16]. Two partner-only extensions build on this model: **tiered subscriptions** (up to five tiers per subscription, each with its own terms, with upgrades and downgrades through `modifySubscription()`) [19] and **add-on subscriptions** (a secondary subscription that is billed separately from, and tied to, a base subscription) [20]. Amazon owns billing, renewal and payment recovery. The developer's server learns about state by calling the Receipt Verification Service (RVS) and, optionally, by receiving Real-Time Notifications (RTN) [12].

**Platform status.** On 2025-02-20 Amazon announced that it would discontinue the Amazon Appstore for Android devices on 2025-08-20. From 2025-02-20, developers could no longer submit new apps that target Android devices, and customers in the JP marketplace could no longer make in-app purchases on those devices. Amazon Coins was discontinued in all marketplaces on the same date [1][3]. Amazon says the change "specifically impacts Android mobile devices only" and that the Appstore continues on Fire TV, Fire Tablet and Fire TV built-in products [1]. For subscriptions, Amazon's FAQ states that active subscribers can keep using their subscriptions on Fire TV, Fire tablets and Fire TV built-in products, or can cancel and "receive a refund for subscriptions that extended past August 20, 2025" [1]. On 2025-08-20, an Amazon staff reply in the Amazon Developer Community clarified that the restriction applies to *non-Amazon* Android devices and that there is no change on Fire TV and Fire tablets [2]. In practice, Subsist should treat Amazon as a Fire OS source. Legacy subscribers from Android phones may appear as cancellations or refunds dated around 2025-08-20.

## APIs and versions

| API / SDK | Current version | What Subsist uses it for | Status |
|---|---|---|---|
| Receipt Verification Service (RVS), `verifyReceiptId` | Operation version `1.0` (in the URL path `/version/1.0/`) [7] | Source of truth for every receipt: status, dates, trial, grace period, cancel reason, promotions, plan changes | Current. Version 1.0 is documented as "the current verifyReceiptId version number" [7][9] |
| RVS Cloud Sandbox | Same operation version `1.0`, under the `/sandbox` path [9] | Verifying App Tester receipts in customer staging environments | Current. It replaced the legacy "RVS Sandbox", which lost support on 2021-03-30 [3] |
| Real-Time Notifications (RTN) | No payload version published. Delivered as Amazon SNS HTTPS messages [12][14] | Push trigger that tells Subsist to re-verify a receipt through RVS | Current (launched 2021-07-15 [3]). Amazon calls it "supplemental"; RVS stays primary [12] |
| Acknowledge Receipt API, `acknowledgeReceipt` | `1.0` [22] | Not needed for ingestion. Useful context: it is a `PUT` that reports the fulfillment result for Quick Subscribe purchases | Current, but Amazon prefers the SDK's `notifyFulfillment()` for new integrations [22] |
| Appstore SDK (client, Java/Android) | 3.0.9, released 2026-05-20. Maven: `com.amazon.device:amazon-appstore-sdk:3.0.9` [4][5] | Not called by Subsist. It is the customer app's source of `userId` and `receiptId`, and of localized `price` via `getProductData()` [21] | Current. 3.0.8 (2025-08-12) and 3.0.7 (2025-04-07) came before it [4] |
| Appstore Billing Compatibility SDK | 4.2.0 (2024-10-14) [3] | None directly. Its receipts are verified through RVS in the same way [3] | No longer supported as of 2026-09-15 [3][6] |
| IAP SDK v2.0 | Legacy | None | Being discontinued; replaced by the Appstore SDK [4][6] |

## Credentials required

| Credential | What it is | Where the customer gets it |
|---|---|---|
| Shared secret | A developer-level secret that "pins an IAP transaction to a particular vendor" and authorizes receipt validation [15]. It goes into the RVS URL path. Production rejects an invalid secret with HTTP 496. RVS Cloud Sandbox accepts any non-empty string [7][9]. | Amazon Developer Console, **Shared Key** page: https://developer.amazon.com/sdk/shared-key.html [7][15]. Amazon recommends calling RVS only from a secure server, never from the app [10]. |
| User ID (`userId` / `appUserId`) | An app-specific ID for one Amazon customer. Recommended storage is 128 characters [15]. | Sent by the customer's app, which reads it from `PurchaseResponse.getUserData().getUserId()` [7]. Appears in RTN as `appUserId` [12]. |
| Receipt ID (`receiptId`) | A globally unique ID for one purchase. Recommended storage is 200 characters [15]. | Sent by the app, which reads it from `PurchaseResponse.getReceipt().getReceiptId()` or `PurchaseUpdatesResponse.getReceipts()` [7]. Appears in every RTN message [12]. |
| App package name | Identifies which app an RTN message belongs to (`appPackageName`) [12]. | The customer's app listing. Subsist uses it to route notifications to the right brand. |
| RTN endpoint registration | A public HTTPS URL, served with a valid certificate from a trusted CA, that accepts HTTPS POST [12]. Amazon subscribes this URL to its own SNS topic. The customer does not create an SNS topic. | Developer Console → App List → select the app → **App Services** → **Real-Time Notifications** → **Add an Endpoint** [13]. Subsist must answer the SNS `SubscribeURL` confirmation before deliveries begin [13]. |

## Server notifications

**Setup.** The customer registers Subsist's HTTPS URL in the Developer Console (App Services → Real-Time Notifications) [13]. Amazon verifies the URL by sending an SNS subscription confirmation message. The endpoint must make an HTTP GET request to the `SubscribeURL` value (the AWS SDK's `DefaultSnsMessageHandler` can do this automatically) [13]. When the endpoint is changed, notifications keep going to the old URL until the new one is verified [13].

**Transport and format.** Each delivery is a standard Amazon SNS JSON envelope (`Type`, `MessageId`, `TopicArn`, `Message`, `Timestamp`, `SignatureVersion`, `Signature`, `SigningCertURL`, `UnsubscribeURL`). The `Message` attribute holds escaped JSON with these fields [12][14]:

| Field | Type | Notes |
|---|---|---|
| `receiptId` | String | Up to 200 characters |
| `relatedReceipts` | Map | For `SUBSCRIPTION_MODIFIED_IMMEDIATE`, contains `cancelledReceiptId`. Otherwise empty (`{}`) [11][12] |
| `appUserId` | String | Up to 128 characters |
| `notificationType` | String | See the table below |
| `appPackageName` | String | Identifies the app |
| `timestamp` | Long | Epoch milliseconds |
| `betaProductTransaction` | Boolean | `true` for Live App Testing purchases |

**Delivery rules.**

- Verify the SNS signature on every message to prevent spoofing [13][25].
- Return HTTP 200. If the server is unreachable or returns a 4xx code, Amazon does not retry. If the server does not answer within 15 seconds, or returns a code outside the 200–4xx range, Amazon counts the delivery as failed and retries [13].
- Ordering is not guaranteed. Use `timestamp` to detect messages that arrive out of order [12].
- Duplicates can occur. Processing must be idempotent [14].
- There is no latency SLA; most notifications publish within seconds [12].
- Amazon may add notification types. Handle unknown types gracefully [12].
- Amazon recommends calling RVS for every notification, because the notification's information may already be out of date [12].

**Notification types → Subsist lifecycle events.** The type names below are copied exactly from Amazon's documentation [12]. The Subsist event is assigned after re-verifying through RVS, because several types need RVS fields to disambiguate.

| `notificationType` | Amazon's meaning [12] | Subsist lifecycle event |
|---|---|---|
| `SUBSCRIPTION_PURCHASED` | A subscription was purchased. | `subscription.started`. Also `trial.started` if RVS `freeTrialEndDate` is non-null. Also `offer.redeemed` if RVS `promotions` is non-null. |
| `SUBSCRIPTION_CONVERTED_FREE_TRIAL_TO_PAID` | An active free trial subscription was converted to paid subscription. | `trial.converted` |
| `SUBSCRIPTION_RENEWED` | An active subscription was renewed. | `payment.renewed` |
| `SUBSCRIPTION_AUTO_RENEWAL_OFF` | Auto-renew feature for subscription was turned off. | `renewal.disabled` |
| `SUBSCRIPTION_AUTO_RENEWAL_ON` | Auto-renew feature for subscription was turned on. | `renewal.enabled` |
| `SUBSCRIPTION_SCHEDULED_TO_END` | Scheduled to end because auto-renew was turned off. Sent 10 days before the end for monthly subscriptions, and 30 days before the end for semi-annual and annual subscriptions. | — (reminder only; `renewal.disabled` was already emitted) |
| `SUBSCRIPTION_IN_GRACE_PERIOD` | A subscription has entered grace period (if enabled). | `grace_period.started` |
| `SUBSCRIPTION_OUT_OF_GRACE_PERIOD` | User fixes the payment issue/failure of the subscription which was previously in grace period. | `payment.renewed` (recovered billing) |
| `SUBSCRIPTION_MODIFIED_DEFERRED` | Plan modified with "deferred" proration; the change takes effect at the next renewal date. | `plan.changed` (scheduled; effective at RVS `deferredDate`, new SKU in `deferredSku`) |
| `SUBSCRIPTION_MODIFIED_IMMEDIATE` | Plan modified with "immediate" proration; the old plan is canceled with a prorated refund and the new plan starts now. Includes the canceled receipt ID. | `plan.changed` (the new `receiptId` replaces `relatedReceipts.cancelledReceiptId`) |
| `SUBSCRIPTION_CANCELLED` | A subscription was voluntarily or involuntarily cancelled. | Decide from RVS `cancelReason`: `1` (customer canceled the order) → `refund.issued`; `2` (canceled by Amazon's system after an unrecovered payment, or by customer support at the customer's request) → `subscription.expired` if `gracePeriodEndDate` had been set, otherwise `revoked`; `4` (replaced by a new subscription) → `plan.changed`; `0` → re-poll later. |
| `SUBSCRIPTION_EXPIRED` | A subscription was expired. | `subscription.expired` |
| `CONSUMABLE_PURCHASED`, `CONSUMABLE_CANCELLED`, `ENTITLEMENT_PURCHASED`, `ENTITLEMENT_CANCELLED` | Non-subscription items were purchased or cancelled. | — (not subscription events; ignore or log) |

Subsist events with **no Amazon equivalent**: `payment.retrying` (Amazon exposes only the grace period), `subscription.on_hold`, `subscription.paused`, `subscription.resumed` and `price.changed`. Amazon documents no account hold, pause or deferral feature for IAP subscriptions, and does not notify on price changes (see below).

## Lifecycle rules and edge cases

**RVS subscription fields.** These are the fields RVS returns for a successful request, as documented [7]:

| Field | Type | Meaning |
|---|---|---|
| `receiptId` | String | Globally unique identifier for the purchase. |
| `productId` | String | The SKU the developer defined. For subscriptions this is the parent SKU [15]. |
| `productType` | String | `CONSUMABLE`, `SUBSCRIPTION` or `ENTITLED`. |
| `termSku` | String | The term (child) SKU. In RVS Cloud Sandbox, `_term` is appended to the parent or base SKU; production does not do this [9]. |
| `term` | String | Duration, for example `1 Week` or `2 Months`. |
| `purchaseDate` | Long (ms) | For subscriptions, the **initial** purchase date, not the date of the latest renewal [7][15]. |
| `renewalDate` | Long (ms) | The date the subscription next needs to renew. It is **null if the subscription is not set to auto-renew** [7]. |
| `cancelDate` | Long (ms) | The date the subscription expired or customer support canceled it. It is null while the receipt is valid. When a customer turns off auto-renew, the cancel date is the date the renewal would have happened [7]. |
| `cancelReason` | Integer | `null` = not canceled; `0` = reason not yet available; `1` = customer canceled the order; `2` = canceled by Amazon's system (for example, an invalid payment not fixed within the grace period, or customer support canceling at the customer's request) [7]; `4` = replaced by a new subscription through a tier or term change [11]. Code `3` is internal to Amazon [11]. |
| `autoRenewing` | Boolean | Whether the subscription will auto-renew (added 2020-09-14) [3][7]. |
| `freeTrialEndDate` | Long (ms) | Set while the subscription is in a free trial; null otherwise. It replaced `isFreeTrial` on 2020-09-21 [3][7]. |
| `gracePeriodEndDate` | Long (ms) | Set while the subscription is in a grace period; null otherwise [7]. |
| `deferredDate`, `deferredSku` | Long (ms), String | For a deferred tier or term change, the date and SKU of the upcoming plan. Both become null once the change takes effect [7][11]. |
| `promotions` | List | A list of `{promotionType, promotionStatus}` objects, or null. `promotionType` is one of `Introductory Price - All Customers`, `Promotional Price - Lapsed Customers` or `Retention Offer`. `promotionStatus` is one of `Queued`, `InProgress` or `Completed` [7]. |
| `fulfillmentDate`, `fulfillmentResult` | Long (ms), String | The developer's fulfillment acknowledgement: `FULFILLED`, `EXISTING_PURCHASE`, `NOT_ELIGIBLE` or `UNAVAILABLE` [7]. |
| `countryCode` | String | The customer's country of residence (added 2025-04-07) [3][7]. |
| `baseReceipts` | List | For an add-on subscription, the receipt IDs of its base subscriptions [7]. |
| `purchaseMetadataMap` | Map | `{"QuickSubscribe":"true"}` for Quick Subscribe purchases; otherwise null [7]. |
| `betaProduct` | Boolean | A Live App Testing product. |
| `testTransaction` | Boolean | A purchase made during Amazon's publishing and testing process. |
| `parentProductId` | String | Always null ("Reserved for future use") [7]. |
| `quantity` | Integer | Always null or 1. |

RVS does **not** return a price, currency or per-renewal transaction ID. The only price the client sees is the localized `price` string from `getProductData()` [21].

**Receipt ID continuity.** A continuous subscription keeps one `receiptId` across all of its renewals. A new receipt appears only after the subscription lapses and the customer subscribes again [15]. In Amazon's example, a subscription active from 2023-01-01 has a `cancelDate` of 2023-03-03. If the customer reactivates it on 2023-04-01, a second receipt is created with a new `purchaseDate` and a null `cancelDate` [7]. An immediate tier change also cancels the old receipt (`cancelReason` 4) and issues a new one, which RTN links through `relatedReceipts.cancelledReceiptId` [11]. Subsist therefore has to **infer renewals** from RTN `SUBSCRIPTION_RENEWED` messages and from `renewalDate` moving forward. It cannot rely on new receipts appearing.

**Renewal dates.** Monthly renewals fall on the same day of the month as the original purchase. When a month is too short, the renewal falls on the closest earlier date. For example, a subscription started January 31 renews on February 28 (or 29), March 31 and April 30 [7].

**Grace period.** RTN sends `SUBSCRIPTION_IN_GRACE_PERIOD` "if enabled" and `SUBSCRIPTION_OUT_OF_GRACE_PERIOD` when the customer fixes the payment [12]. RVS exposes `gracePeriodEndDate` [7]. If payment is still not fixed when the grace period ends, the subscription is canceled with `cancelReason` 2 [7]. The length of the grace period is not stated in current documentation. A legacy Amazon Appstore blog post said "a maximum of six days", but that post now redirects, so the figure is **(unverified)** [26].

**Account hold and pause.** Neither is documented for Amazon IAP subscriptions. Amazon has no notification types or RVS fields for them [7][12].

**Free trials.** A term can offer a trial of 7 days, 14 days, 1 month, 2 months or 3 months [16]. The trial runs *in addition to* the paid term, and billing starts when it ends. If the customer turns off auto-renew during the trial, the subscription ends without a charge [15]. Each customer gets one trial per subscription product, even if the earlier trial was cut short [15]. Trial conversion arrives as `SUBSCRIPTION_CONVERTED_FREE_TRIAL_TO_PAID` [12].

**Introductory and promotional pricing.** Promotional pricing is either a one-time upfront price covering *n* terms, or a recurring discount for up to *n* terms. The combined promotional duration can be at most 12 weeks for weekly and bi-weekly terms, or 12 months for other terms. An offer targets either all customers or only lapsed customers. It is available worldwide except in the IN marketplace [17]. If a product has both a trial and promotional pricing, the promotion starts after the trial ends (RVS `promotionStatus` stays `Queued` during the trial) [7][15]. A customer can use promotional pricing again only after being unsubscribed for 12 or more months, and promotional pricing cannot be combined with other discounts [17]. Promotion details appear only on the receipt for the purchase that used the promotion [7].

**Retention offers.** A customer who starts cancelling on Amazon's retail website can be offered a percentage discount (a whole number from 1 to 100) that applies from the next renewal. The maximum duration is 3 terms for weekly, bi-weekly and monthly subscriptions, 2 for bi-monthly and quarterly, and 1 for semi-annual and annual. To be eligible, the customer must be on an active full-price subscription and must not have used a retention offer in the last 12 months [18]. RVS reports these offers as `promotionType` `Retention Offer`. After the offer ends, the details disappear from the receipt [7]. Map a redeemed retention offer to `offer.redeemed`.

**Price changes.** When a price is raised, existing subscribers, including customers in a free trial, keep paying their original price; only new subscribers pay more. When a price is lowered, existing subscribers who are paying more move to the new price at their next renewal [16]. No RTN notification is sent. A `price.changed` event could only be inferred from outside data, so none is emitted.

**Upgrades and downgrades.** These exist only for tiered subscriptions, which require partner enablement and SDK 3.0.7 or later [19]. An *immediate* change cancels the old receipt with a prorated refund and starts a new receipt. A *deferred* change keeps the current receipt and sets `deferredDate` and `deferredSku` until the new plan takes effect [11]. A customer can hold only one tier or term at a time [19].

**Cancellations and refunds.** Turning off auto-renew does not end access; the customer keeps access until the end of the paid term [23]. Pro-rated refunds go through Amazon customer service [15][23]. RVS returns HTTP 410 for a receipt that "is no longer valid. Treat it as a canceled receipt" [7]. Since 2025-05-29, sending `UNAVAILABLE` through `notifyFulfillment()` immediately cancels and refunds the purchase. Since SDK 3.0.9, `EXISTING_PURCHASE` and `NOT_ELIGIBLE` do the same [3][22]. For Quick Subscribe (a partner-only feature), a receipt that is not marked `FULFILLED` within 14 days is automatically canceled and refunded [22]. Canceling a base subscription automatically cancels its add-on subscriptions [20].

**Test data.** Live App Testing purchases carry `betaProduct: true` in RVS and `betaProductTransaction: true` in RTN [7][12]. App Tester receipts can only be verified against RVS Cloud Sandbox [9]. Accelerated test subscriptions renew every 5 to 30 minutes depending on the term, and are canceled after four renewals [24].

## Checking current status (re-sync)

RVS is a GET request against a fixed path [7][9]. Amazon publishes working sample requests and responses in [8]:

```
Production: https://appstore-sdk.amazon.com/version/1.0/verifyReceiptId/developer/{shared-secret}/user/{user-id}/receiptId/{receipt-id}
Sandbox:    https://appstore-sdk.amazon.com/sandbox/version/1.0/verifyReceiptId/developer/{shared-secret}/user/{user-id}/receiptId/{receipt-id}
```

| HTTP code | Meaning [7] | Subsist handling |
|---|---|---|
| 200 | Valid; the body holds receipt JSON | Diff the result against the stored state and emit events |
| 400 | Invalid receipt, or no transaction found | Flag the receipt; do not retry |
| 410 | Receipt no longer valid; treat it as canceled | Mark the receipt `revoked` or `subscription.expired` |
| 429 | Throttled | Back off and retry |
| 496 | Invalid shared secret | Mark the credential as broken and alert the customer |
| 497 | Invalid user ID | Flag the receipt |
| 500 | Internal server error | Retry with backoff |

**How to poll.** RVS verifies one receipt at a time. There is no endpoint that lists a user's receipts [7]. Subsist can only re-sync receipts it already knows. New receipts arrive from the customer's app, which gets the full history from `getPurchaseUpdates(true)` [21], or from RTN messages [12]. Recommended practice for Subsist:

1. Re-verify immediately after every RTN message [12].
2. Re-verify each active receipt shortly after its `renewalDate`, and after `gracePeriodEndDate` or `deferredDate` when those are set.
3. If `cancelReason` is `0`, poll again later, because the reason "will render at a later time" [7].
4. Run a slow sweep of all non-expired receipts as a safety net.

Amazon does not publish a rate limit, so any request rate is unverified. Honor HTTP 429.

## Migration notes for subsist-app

This section compares what `subsist-app/amazon/models.py` (2022) does today with what it should do.

- **Endpoint is still correct.** The code calls the production RVS `https://appstore-sdk.amazon.com/version/1.0/verifyReceiptId/...` URL, and 1.0 is still the current operation version [7]. The remaining gaps: add a sandbox mode (`/sandbox/version/1.0/...`) for App Tester receipts [9]; add a request timeout; and keep the shared secret out of logs and exception messages, since it sits in the URL path.
- **Error handling.** `raise_for_status()` treats every non-200 response the same way. Handle 410 as "canceled", 429 and 500 as retryable, 496 as a credentials failure, and 400 and 497 as bad input [7].
- **Timestamp bug.** `datetime.fromtimestamp(ts / 1000).replace(tzinfo=pytz.utc)` reads the epoch in the *server's local* time zone and then relabels it as UTC. Use `datetime.fromtimestamp(ts / 1000, tz=datetime.timezone.utc)`.
- **Null `renewalDate` crash.** `renewalDate` is null when auto-renew is off [7], so `from_timestamp(None)` raises an error for every subscriber who has turned off renewal. Guard every date field.
- **`renewal_status` is derived from the wrong field.** The code uses `cancelDate`, but a non-null `cancelDate` means the subscription has expired or was canceled, not that renewal is off. Use `autoRenewing` for renewal status, and use `cancelDate` together with `cancelReason` for termination [7].
- **`in_free_trial="unknown"`.** Use `freeTrialEndDate` (null means not in a trial) [7].
- **Billing cycle.** `billing_cycle_start = purchaseDate` is correct only for the first period, because `purchaseDate` is the *initial* purchase date [7][15]. Derive each cycle from successive `renewalDate` values and `term`.
- **Only one transaction is ever recorded.** `transaction_id` and `subscription_id` are both set to `receiptId`, and `get_or_create` is called on them. Because a continuous subscription keeps one `receiptId` [15], renewals are never recorded. Record a renewal whenever `renewalDate` advances or RTN reports `SUBSCRIPTION_RENEWED`, using a synthetic ID such as `receiptId` plus the cycle start. Link receipt chains by `appUserId` and parent `productId`, and through `relatedReceipts.cancelledReceiptId` or `cancelReason` 4 for tier changes [11].
- **Test data handling.** Dropping every `betaProduct` or `testTransaction` receipt hides sandbox and Live App Testing data. Store these receipts with an environment flag instead. Also read the fields with `.get()`, since `KeyError` is possible on unexpected payloads.
- **Price and currency (`"n/a"`).** RVS still returns no price [7]. Store `countryCode` (available since 2025-04-07) [3]. Take the amount from the localized `price` string the client app sends from `getProductData()` [21], or from Amazon sales reports (not yet evaluated).
- **New fields to persist:** `autoRenewing`, `cancelReason`, `freeTrialEndDate`, `gracePeriodEndDate`, `deferredDate`, `deferredSku`, `promotions`, `fulfillmentResult`, `fulfillmentDate`, `countryCode`, `baseReceipts`, `purchaseMetadataMap` and `productId` (the parent SKU) [7].
- **Add RTN ingestion.** No RTN handling exists yet. Add an HTTPS endpoint that verifies SNS signatures, confirms `SubscribeURL`, dedupes on `MessageId`, orders messages by `timestamp`, returns 200 within 15 seconds, and then calls RVS [12][13][14].
- **Platform scope.** Label Amazon as Fire OS only (Fire TV, Fire tablets, Fire TV built-in) [1]. Expect cancellations and refunds around 2025-08-20 for legacy subscribers on Android phones [1].

## References

1. Upcoming changes to Amazon Appstore for Android devices and other programs (Amazon Developer blog, 2025-02-20) — https://developer.amazon.com/apps-and-games/blogs/2025/02/upcoming-changes-to-amazon-appstore-for-android-devices-and-coins-program (accessed 2026-09-28)
2. What's changed as of Aug 20? Amazon Appstore for Android shutdown (Amazon Developer Community Q&A, Amazon staff reply, 2025-08-20) — https://community.amazondeveloper.com/t/whats-changed-as-of-aug-20-amazon-appstore-for-android-shutdown/16282 (accessed 2026-09-28)
3. In-App Purchasing Release Notes — https://developer.amazon.com/docs/in-app-purchasing/iap-whats-new.html (accessed 2026-09-28)
4. Appstore SDK Release Notes — https://developer.amazon.com/docs/appstore-sdk/release-notes.html (accessed 2026-09-28)
5. Integrate the Appstore SDK — https://developer.amazon.com/docs/appstore-sdk/integrate-appstore-sdk.html (accessed 2026-09-28)
6. SDKs (Amazon Appstore Developers download page) — https://developer.amazon.com/apps-and-games/sdk-download-notes (accessed 2026-09-28)
7. Receipt Verification Service for Appstore SDK IAP — https://developer.amazon.com/docs/in-app-purchasing/iap-rvs-for-android-apps.html (accessed 2026-09-28)
8. RVS Examples for Appstore SDK IAP — https://developer.amazon.com/docs/in-app-purchasing/iap-rvs-examples.html (accessed 2026-09-28)
9. Use RVS Cloud Sandbox — https://developer.amazon.com/docs/in-app-purchasing/rvs-cloud-sandbox.html (accessed 2026-09-28)
10. RVS Production Setup for Appstore SDK IAP — https://developer.amazon.com/docs/in-app-purchasing/iap-rvs-setup-prod.html (accessed 2026-09-28)
11. Receipt Verification for Tiered Subscriptions — https://developer.amazon.com/docs/in-app-purchasing/iap-rvs-for-tiered-subs.html (accessed 2026-09-28)
12. Understanding Real-Time Notifications — https://developer.amazon.com/docs/in-app-purchasing/real-time-notifications.html (accessed 2026-09-28)
13. Use Real-Time Notifications — https://developer.amazon.com/docs/in-app-purchasing/use-rtn.html (accessed 2026-09-28)
14. Real-Time Notifications Examples — https://developer.amazon.com/docs/in-app-purchasing/rtn-example.html (accessed 2026-09-28)
15. In-App Purchasing FAQ — https://developer.amazon.com/docs/in-app-purchasing/iap-faqs.html (accessed 2026-09-28)
16. Create and Submit Single IAP items — https://developer.amazon.com/docs/in-app-purchasing/iap-create-and-submit-iap-items.html (accessed 2026-09-28)
17. Set Up Promotional Pricing — https://developer.amazon.com/docs/in-app-purchasing/promotional-pricing.html (accessed 2026-09-28)
18. Retention Offers — https://developer.amazon.com/docs/reports-promo/retention-offers.html (accessed 2026-09-28)
19. Tiered Subscriptions Overview — https://developer.amazon.com/docs/in-app-purchasing/tiered-subscriptions-overview.html (accessed 2026-09-28)
20. Add-On Subscriptions Overview — https://developer.amazon.com/docs/in-app-purchasing/add-on-subscriptions-overview.html (accessed 2026-09-28)
21. Implement Appstore SDK IAP — https://developer.amazon.com/docs/in-app-purchasing/iap-implement-iap.html (accessed 2026-09-28)
22. Set Up Quick Subscribe (includes the Acknowledge Receipt API and auto-cancellation) — https://developer.amazon.com/docs/in-app-purchasing/set-up-quick-subscribe.html (accessed 2026-09-28)
23. Types of Purchases — https://developer.amazon.com/docs/in-app-purchasing/purchase-types.html (accessed 2026-09-28)
24. Accelerated Subscriptions Time Table — https://developer.amazon.com/docs/app-testing/accelerated-subscriptions-time-table.html (accessed 2026-09-28)
25. Verifying the signatures of Amazon SNS messages (AWS documentation) — https://docs.aws.amazon.com/sns/latest/dg/sns-verify-signature-of-message.html (accessed 2026-09-28; linked from [13], not fetched directly)
26. Understanding IAP Subscription Behavior (legacy Amazon Appstore blog; the URL now redirects to the blog index) — https://developer.amazon.com/blogs/appstore/post/75c6b03a-71ad-4008-985b-8da3ea7da902/understanding-the-iap-subscription-behavior (accessed 2026-09-28)
