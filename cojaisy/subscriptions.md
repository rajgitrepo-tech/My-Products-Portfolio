# Cojaisy — Subscription & Monetisation

## Subscription Tiers

| Feature | Free | Wellness ₹99/mo | FamilyCare ₹499/mo |
|---------|------|-----------------|-------------------|
| Voice queries/month | 2 | 100 | 500 (shared) |
| Food photo recognition/month | 2 | 5 | 50 (shared) |
| AI recipe generation/month | 2 | 20 | 100 (shared) |
| Manual food logging | Unlimited | Unlimited | Unlimited |
| MyShelf (pantry) | Basic | AI-powered | AI-powered |
| Meal planning | Basic | Full | Full |
| Health insights | Basic | Full | Full |
| Voice quality | Standard | Neural | Neural |
| Family members | 1 | 1 | Up to 5 |
| Family dashboard | ✗ | ✗ | ✅ |
| Shared shopping list | ✗ | ✗ | ✅ |
| **Cost to serve** | ~₹8 | ~₹41 | ~₹267 |
| **Margin** | — | **₹58** | **₹232** |

---

## Payment Architecture

### Technology
- **Payment Gateway:** Razorpay (India-first, supports UPI, cards, netbanking)
- **Model:** Recurring subscriptions (auto-debit monthly)
- **Flutter SDK:** razorpay_flutter ^1.3.6
- **Backend:** subscription-handler Lambda + Razorpay webhooks

### Subscription Lifecycle

```
CREATE
  User selects tier
  → POST /subscriptions/create
  → subscription-handler creates Razorpay subscription (plan_id)
  → Returns: subscription_id, razorpay_key_id, amount
  → Flutter opens Razorpay checkout (recurring: true)
  → User authorises monthly charge

VERIFY
  Razorpay returns: payment_id, subscription_id, signature
  → POST /subscriptions/verify
  → HMAC-SHA256 signature verification
  → DynamoDB: user tier updated, usage counters reset
  → User gains tier access immediately

RECURRING (monthly)
  Razorpay auto-charges on billing date
  → Webhook: subscription.charged
  → subscription-handler: reset usage counters, extend end_date
  → User continues uninterrupted

CANCEL
  User taps "Cancel Subscription"
  → POST /subscriptions/cancel
  → Razorpay stops future charges
  → User retains tier until current period ends
  → Webhook: subscription.cancelled → DynamoDB status updated

PAYMENT FAILURE
  Razorpay retries 3-5 times
  → Webhook: subscription.payment_failed (each attempt)
  → Webhook: subscription.paused (after retries exhausted)
  → subscription-handler: downgrade user to free tier
  → CloudWatch metric: PaymentFailure
```

### Webhook Events Handled

| Event | Action |
|-------|--------|
| `subscription.charged` | Reset usage counters, extend subscription |
| `subscription.cancelled` | Mark cancelled, keep tier until end_date |
| `subscription.paused` | Downgrade to free tier immediately |
| `subscription.payment_failed` | Log to CloudWatch, notify (future) |

Webhook security: HMAC-SHA256 signature verification using webhook secret
stored in AWS Secrets Manager.

---

## Usage Tracking

Every AI feature call checks and increments usage counters:

```javascript
// Before AI call
const user = await dynamodb.get({ TableName: 'kiro-users-prod', Key: { user_id } });
const { voice_count_month, subscription_tier, usage_reset_date } = user;

// Check if monthly reset needed
if (isNewMonth(usage_reset_date)) {
  await resetUsageCounters(user_id);
}

// Check limit
const limit = TIER_LIMITS[subscription_tier].voice_count;
if (voice_count_month >= limit) {
  return { statusCode: 429, body: 'Monthly voice limit reached. Upgrade to continue.' };
}

// Increment counter
await dynamodb.update({
  TableName: 'kiro-users-prod',
  Key: { user_id },
  UpdateExpression: 'SET voice_count_month = voice_count_month + :inc',
  ExpressionAttributeValues: { ':inc': 1 }
});
```

Usage counters reset on the 1st of each month (or on subscription renewal).

---

## Admin Subscription Analytics

The admin dashboard provides real-time subscription metrics:

- Total users by tier (Free / Wellness / FamilyCare)
- Monthly Recurring Revenue (MRR)
- Churn rate (cancellations / active subscriptions)
- Payment failure rate
- AI usage per tier (voice, photo, recipe counts)
- Cost per user per tier
- Revenue vs infrastructure cost comparison

All metrics sourced from DynamoDB aggregations via admin-handler Lambda,
with CloudWatch custom metrics for payment events.
