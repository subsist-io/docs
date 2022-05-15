# Subscription Lifecycle

The subscription lifecycle is a finite state machine (FSM), as subscription can only be in one state at a time.

The steps in the lifecycle are generally:
- User creates a subscription, entering a free trial
- User converts to paid subscriber when free trial ends and payment is made
- User subscription renews, user makes an additional payment
- User cancels auto renew for their subscription, meaning User still has service, but will churn
- User's subscription expires at the end of the service period, user churns out of active service

The trick is knowing which of those actions the user has taken, and what has happened to the subscription in the mean time.

Each of the Merchants have different names for these events, and each of the merchants has their own special edge cases to account for. 

Here are some examples:
- There are extended trials, and introductory pricing options
- Google has Subscription Pause where the sub is on hold. (For how long, and how to re-enable or not)
- Apple will retry payments for up to 60 days
- 

Note:
- users can have multiple subscriptions at the same time
- users can have multiple receipts at the same time
- people using your app can have multiple user accounts
- multiple users of your can present the same receipt or subscription information from the store
- you can have a subscription and not know the user
- you can have a receipt and not know the user 

The questions are, what can first time payment developers to track this, and how important is this to the success of your business?
- Not enough
- Extremely 