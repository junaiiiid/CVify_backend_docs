# Flutter Integration — Subscriptions & In-App Purchases (Android + iOS)

This guide covers everything the backend now supports for billing on **both
Google Play (Android) and the App Store (iOS)**. Hand this to the Flutter
developer responsible for the billing/purchase flow.

---

## What was built

| Component | Description |
|---|---|
| `verifyPurchase` (updated) | Now cross-platform via a `platform` request field. Verifies Android purchases through Google Play Android Publisher API and iOS through Apple's App Store Server API. Writes credits + subscription state to DB. |
| `handleSubscriptionRenewal` | Android webhook — called by Google Play via Pub/Sub on renewal/expire/refund. Flutter never calls this. |
| `handleAppleNotification` | iOS webhook — called by Apple via App Store Server Notifications V2 on renewal/expire/refund. Flutter never calls this. |
| `getCredits` (unchanged) | Already returns `subscription_tier`, `subscription_expires_at`, `is_subscribed`. Single source of truth on both platforms. |

---

## Store-configured product IDs

The backend recognises these product IDs. **Both stores use the same IDs except
for one (`cvify_pro` on iOS vs `cvify_pro_monthly` on Android)** — keep this in
mind when looking up the product in your `in_app_purchase` integration.

### Subscriptions (`purchase_type: "subs"`)

| Product ID | Platform | Credits granted | Tier |
|---|---|---|---|
| `cvify_pro_monthly` | Android | 25 | `"pro"` |
| `cvify_pro` | iOS | 25 | `"pro"` |
| `cvify_max_monthly` | Both | 50 | `"max"` |

### One-time consumables (`purchase_type: "inapp"`)

| Product ID | Platform | Credits granted |
|---|---|---|
| `cvify_credits_25` | Both | 25 |
| `cvify_credits_50` | Both | 50 |
| `cvify_credits_75` | Both | 75 |
| `cvify_credits_100` | Both | 100 |

---

## `POST /verifyPurchase`

Call this immediately after **every successful purchase** — both subscriptions
and one-time packs, on both platforms.

### Request shape

```json
{
  "purchase_token": "<see below>",
  "product_id":     "cvify_credits_50",
  "purchase_type":  "inapp",       // or "subs"
  "platform":       "ios"          // or "android" — defaults to "android"
}
```

### What goes in `purchase_token`?

| Platform | Value to send |
|---|---|
| **Android** | The token from `GooglePlayPurchaseDetails.purchaseToken` (or `PurchaseDetails.verificationData.serverVerificationData` on Android) |
| **iOS** | The StoreKit transaction ID — from `AppStorePurchaseDetails.purchaseID` on the `in_app_purchase` plugin, or `transactionId` if you're using StoreKit2 directly |

The backend uses this value as a globally-unique purchase identifier and stores
it for replay protection.

### Response shape (same on both platforms)

```json
{
  "status_code": 200,
  "status": "success",
  "data": {
    "credits": 37,
    "credits_added": 25,
    "product_id": "cvify_pro",
    "subscription_tier": "pro",
    "subscription_expires_at": "2026-07-05T09:00:00.000Z"
  }
}
```

- `subscription_tier` — `"pro"` | `"max"` | `null`. `null` for one-time purchases.
- `subscription_expires_at` — ISO 8601 timestamp. `null` for one-time purchases.

### Dart model (unchanged from before)

```dart
class VerifyPurchaseResponse {
  final int credits;
  final int creditsAdded;
  final String productId;
  final String? subscriptionTier;
  final DateTime? subscriptionExpiresAt;

  factory VerifyPurchaseResponse.fromJson(Map<String, dynamic> json) {
    final d = json['data'] as Map<String, dynamic>;
    return VerifyPurchaseResponse(
      credits:                d['credits'] as int,
      creditsAdded:           d['credits_added'] as int,
      productId:              d['product_id'] as String,
      subscriptionTier:       d['subscription_tier'] as String?,
      subscriptionExpiresAt:  d['subscription_expires_at'] != null
          ? DateTime.parse(d['subscription_expires_at'] as String)
          : null,
    );
  }
}
```

### Error codes (same on both platforms)

| HTTP | code | Meaning |
|---|---|---|
| 409 | `ALREADY_REDEEMED` | Token already used — safe to ignore, purchase was already credited. Still call `completePurchase()`. |
| 400 | — | Unknown `product_id`, invalid `purchase_type`, or invalid `platform` |
| 400 | `INVALID_PURCHASE_STATE` | Purchase was not in a valid state (refunded, revoked, bundleId mismatch, expired sub, etc.) |
| 502 | — | Store verification call to Google or Apple failed |

---

## Recommended Flutter integration

The flow is identical on both platforms — only `platform` and `purchase_token`
vary. Pseudocode using the `in_app_purchase` package:

```dart
Future<void> onPurchaseSuccess(PurchaseDetails purchase) async {
  // 1. Map platform + token
  final isIos = Platform.isIOS;
  final String purchaseToken;
  if (isIos) {
    // StoreKit transaction ID
    purchaseToken = purchase.purchaseID!;
  } else {
    // Google Play token — on Android, in_app_purchase exposes this via the
    // GooglePlayPurchaseDetails subclass or via verificationData.serverVerificationData
    purchaseToken = (purchase as GooglePlayPurchaseDetails).billingClientPurchase.purchaseToken;
    // alternatively: purchase.verificationData.serverVerificationData
  }

  // 2. Classify product
  final isSub = purchase.productID == 'cvify_pro_monthly'
              || purchase.productID == 'cvify_pro'
              || purchase.productID == 'cvify_max_monthly';

  // 3. Verify with backend
  final response = await supabase.functions.invoke(
    'verifyPurchase',
    body: {
      'purchase_token': purchaseToken,
      'product_id':     purchase.productID,
      'purchase_type':  isSub ? 'subs' : 'inapp',
      'platform':       isIos ? 'ios' : 'android',
    },
  );

  // 4. Apply response to local state
  final data = response.data['data'];
  state.credits              = data['credits'];
  state.subscriptionTier     = data['subscription_tier'];
  state.subscriptionExpiresAt = data['subscription_expires_at'] != null
      ? DateTime.parse(data['subscription_expires_at'])
      : null;

  // 5. Mark the purchase as complete with the store
  if (purchase.pendingCompletePurchase) {
    await InAppPurchase.instance.completePurchase(purchase);
  }
}
```

### Critical: always call `completePurchase()`

On **both platforms**, you must call `completePurchase()` after a successful
backend verification. Without this:
- **Google Play** will retry the purchase and the user may be re-charged or see
  the consumable as unredeemed
- **Apple** will keep the transaction in the pending queue and replay it on
  every app launch

Even on `ALREADY_REDEEMED` (409) response, call `completePurchase()` — the
credits are already on the user, but the store-side state needs closing.

---

## Subscription renewals — Flutter does nothing

Both stores send renewal events directly to backend webhooks:

- **Google Play** → Cloud Pub/Sub → `/handleSubscriptionRenewal`
- **Apple** → App Store Server Notifications V2 → `/handleAppleNotification`

When a renewal fires, the backend:
1. Grants the monthly credits to the user
2. Extends `subscription_expires_at` by ~31 days (or Apple's reported `expiresDate`)
3. The next time the app calls `getCredits`, the new state is reflected

On cancel/expire/refund, the backend clears `subscription_tier` and
`subscription_expires_at`.

**No Flutter code needed for renewals** — just refresh `getCredits` on app
start and after any purchase.

---

## Subscription state in the app

```dart
final credits = await api.getCredits();

final bool isPaid = credits.isSubscribed;

final String badge = switch (credits.subscriptionTier) {
  'pro' => 'Pro',
  'max' => 'Max',
  _     => '',
};

final DateTime? renewsOn = credits.subscriptionExpiresAt;
```

Drive UI off `is_subscribed` (single bool) and `subscription_tier` (badge
label). `subscription_expires_at` is purely informational ("Your plan renews
on Jul 5").

---

## Testing

### Android (sandbox)
1. Add your test Google account as a license tester in Google Play Console →
   Setup → License testing
2. Same account must be on the device
3. Internal/closed track APK installed via Play Store
4. Make a purchase — you won't be charged

### iOS (sandbox)
1. Add a sandbox tester in App Store Connect → Users and Access → Sandbox
2. On the iPhone: **Settings → App Store → Sandbox Account** → sign in with
   the tester account
3. Run the iOS build (TestFlight or local install)
4. Make a purchase — you won't be charged

Backend logs (`verifyPurchase`, `handleAppleNotification`,
`handleSubscriptionRenewal`) show every step including the resolved product ID,
credit grant, and subscription update.

---

## Summary checklist

What needs to change in the Flutter app:

- [ ] Update billing call to send `platform: "ios" | "android"` to `/verifyPurchase`
- [ ] On iOS, send `purchase.purchaseID` (StoreKit transaction ID) as `purchase_token`
- [ ] On Android, send `googlePlayPurchase.billingClientPurchase.purchaseToken` (or `verificationData.serverVerificationData`) as `purchase_token`
- [ ] Map iOS product `cvify_pro` to the Pro tier when displaying — same data flow as Android's `cvify_pro_monthly`
- [ ] Call `completePurchase()` on success **and** on `ALREADY_REDEEMED`
- [ ] Refresh `getCredits` on app start and after any purchase — that's how renewals/expiries reach the UI
