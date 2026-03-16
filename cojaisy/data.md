# Data Architecture — Deep Dive

> 12 DynamoDB tables. Zero idle cost. Every schema decision made for query efficiency and user isolation.

[![DynamoDB](https://img.shields.io/badge/Database-Amazon_DynamoDB-4053D6?style=flat&logo=amazondynamodb&logoColor=white)](https://aws.amazon.com/dynamodb)
[![S3](https://img.shields.io/badge/Storage-Amazon_S3-FF9900?style=flat&logo=amazonaws&logoColor=white)](https://aws.amazon.com/s3)

---

## Design Principles

- **On-demand capacity** — zero provisioned throughput, zero idle cost, auto-scales to any spike
- **User-scoped partition keys** — `user_id` as PK on every table ensures data isolation at the storage layer
- **TTL for ephemeral data** — OTP tokens auto-deleted by DynamoDB, no cleanup Lambda needed
- **GSIs for access patterns** — phone number lookup, conversation history, recipe browsing
- **Minimal item size** — only store what the application needs, no bloated documents

---

## Table Schemas

### kiro-users
The central user record — health profile, subscription state, usage counters.

```
user_id (PK)              ← Cognito sub (UUID)
phone_number              ← GSI PK for phone-based lookups
name
age
gender
height_cm
weight_kg
bmi                       ← calculated, stored for AI context
dietary_preference        ← vegetarian | egg | non-veg | vegan
health_goals              ← weight_loss | muscle_gain | maintenance
allergies                 ← JSON array
locality                  ← city/region for regional cuisine context
subscription_tier         ← free | wellness | familycare
subscription_start_date
subscription_end_date
razorpay_subscription_id
voice_count_month         ← AI usage tracking (reset monthly)
photo_count_month
recipe_count_month
usage_reset_date
created_at
updated_at
```

GSI: `phone_number-index` — enables Cognito custom auth challenge to look up user by phone.

---

### kiro-auth-tokens
Ephemeral OTP tokens. DynamoDB TTL handles cleanup automatically.

```
token_id (PK)             ← UUID
phone_number              ← GSI PK
otp_code                  ← 6-digit code (hashed)
expires_at                ← TTL attribute — DynamoDB auto-deletes after 5 minutes
attempts                  ← incremented on each wrong attempt (max 3)
verified                  ← boolean — single-use enforcement
created_at
```

TTL on `expires_at` means no cleanup Lambda, no cron job, no stale tokens.

---

### kiro-conversations
AI chat session metadata.

```
conversation_id (PK)      ← UUID
user_id                   ← GSI PK
created_at
last_message_at
message_count
title                     ← auto-generated from first message
```

GSI: `user_id-index` — fetch all conversations for a user, sorted by `last_message_at`.

---

### kiro-messages
Full conversation history — the context window for AI.

```
message_id (PK)           ← UUID
conversation_id           ← GSI PK
user_id
role                      ← user | assistant
content                   ← message text
timestamp
token_count               ← tracked for context window management
```

GSI: `conversation_id-index` — fetch last N messages for context assembly.
Query pattern: `conversation_id = X ORDER BY timestamp DESC LIMIT 10`

---

### kiro-recipes
Recipe library — user-created, AI-generated, and pre-seeded.

```
recipe_id (PK)            ← UUID
user_id                   ← GSI PK
name
description
prep_time_minutes
cook_time_minutes
servings
ingredients               ← JSON array [{item, quantity, unit}]
instructions              ← JSON array [step strings]
nutrition_per_serving     ← JSON {calories, protein, carbs, fat}
dietary_type              ← vegetarian | egg | non-veg | vegan
cuisine                   ← North Indian | South Indian | Bengali | etc.
difficulty                ← easy | medium | hard
is_ai_generated           ← boolean
is_default                ← boolean (pre-seeded recipes)
tags                      ← JSON array
created_at
```

GSI: `user_id-index` — fetch user's recipe library.

---

### kiro-meal-plans
Daily meal planning records.

```
plan_id (PK)              ← UUID
user_id                   ← GSI PK
date                      ← YYYY-MM-DD
meal_type                 ← breakfast | lunch | dinner | snack
recipe_id                 ← reference to kiro-recipes
meal_name                 ← display name
nutrition                 ← JSON {calories, protein, carbs, fat}
servings
notes
created_at
```

GSI: `user_id-date-index` — fetch all meals for a user on a specific date.

---

### kiro-nutrition-logs
Daily food intake records — manual entry and photo-based.

```
log_id (PK)               ← UUID
user_id                   ← GSI PK
date                      ← YYYY-MM-DD (GSI SK)
food_name
calories
protein_g
carbs_g
fat_g
serving_size
serving_unit
meal_type                 ← breakfast | lunch | dinner | snack
source                    ← manual | photo | recipe
photo_s3_key              ← if source = photo
created_at
```

GSI: `user_id-date-index` — fetch all logs for a user on a date (today's nutrition for AI context).

---

### kiro-shelf-items
Pantry inventory — the live input to AI recipe suggestions.

```
item_id (PK)              ← UUID
user_id                   ← GSI PK
name
quantity
unit                      ← kg | g | L | ml | pieces | packets
category                  ← vegetables | grains | dairy | spices | etc.
expiry_date               ← YYYY-MM-DD (used for expiry alerts in AI context)
added_date
notes
```

GSI: `user_id-index` — fetch user's full shelf inventory.
Query pattern for AI context: `user_id = X AND expiry_date < today + 7` (expiring items).

---

### kiro-subscriptions
Billing records and subscription lifecycle.

```
subscription_id (PK)      ← UUID
user_id                   ← GSI PK
razorpay_subscription_id  ← Razorpay reference
tier                      ← wellness | familycare
status                    ← created | active | cancelled | paused
amount
currency
start_date
end_date
next_billing_date
cancelled_at
admin_note                ← for manual overrides
created_at
updated_at
```

---

### kiro-alarms
Wellness reminder schedules.

```
alarm_id (PK)             ← UUID
user_id                   ← GSI PK
title
message
scheduled_time            ← HH:MM
days_of_week              ← JSON array [0-6]
is_active                 ← boolean
notification_id           ← Android local notification ID
created_at
```

---

## S3 Storage Structure

```
{bucket-name}/
├── food-photos/
│   └── {user_id}/
│       └── {timestamp}.jpg          ← food recognition photos
├── voice-input/
│   └── {user_id}/
│       └── {timestamp}.m4a          ← voice query audio
├── voice-cache/
│   └── {sha256_hash}.mp3            ← cached Polly responses (24h TTL)
└── transcripts/
    └── {job_id}.json                ← Transcribe output (temp)
```

All objects accessed via pre-signed URLs (5-minute expiry). No public access.

---

## Context Assembly Query Pattern

The AI context pipeline runs these queries in parallel before every chat message:

```javascript
const [user, shelfItems, expiringItems, todayNutrition, recentMeals, messages] =
  await Promise.all([
    // User profile
    dynamodb.get({ TableName: USERS_TABLE, Key: { user_id } }),

    // Full shelf inventory
    dynamodb.query({
      TableName: SHELF_ITEMS_TABLE,
      KeyConditionExpression: 'user_id = :uid',
      ExpressionAttributeValues: { ':uid': userId },
      Limit: 20
    }),

    // Items expiring in next 7 days
    dynamodb.query({
      TableName: SHELF_ITEMS_TABLE,
      KeyConditionExpression: 'user_id = :uid',
      FilterExpression: 'expiry_date BETWEEN :today AND :future',
      ExpressionAttributeValues: {
        ':uid': userId,
        ':today': today,
        ':future': sevenDaysFromNow
      }
    }),

    // Today's nutrition totals
    dynamodb.query({
      TableName: NUTRITION_LOGS_TABLE,
      IndexName: 'user_id-date-index',
      KeyConditionExpression: 'user_id = :uid AND #date = :today',
      ExpressionAttributeValues: { ':uid': userId, ':today': today }
    }),

    // Recent meal history (3 days)
    dynamodb.query({
      TableName: MEAL_PLANS_TABLE,
      IndexName: 'user_id-date-index',
      KeyConditionExpression: 'user_id = :uid AND #date >= :threeDaysAgo',
      ExpressionAttributeValues: { ':uid': userId, ':threeDaysAgo': threeDaysAgo }
    }),

    // Last 10 messages (conversation history)
    dynamodb.query({
      TableName: MESSAGES_TABLE,
      IndexName: 'conversation_id-index',
      KeyConditionExpression: 'conversation_id = :cid',
      ExpressionAttributeValues: { ':cid': conversationId },
      ScanIndexForward: false,
      Limit: 10
    })
  ]);
```

Six parallel DynamoDB reads assembled into a single dynamic system prompt — all before Bedrock is invoked.

---

## Data Retention & Lifecycle

| Data Type | Retention | Mechanism |
|-----------|-----------|-----------|
| OTP tokens | 5 minutes | DynamoDB TTL |
| Food photos | 90 days standard, then Glacier | S3 lifecycle rule |
| Voice input audio | 24 hours | S3 lifecycle rule |
| Voice response cache | 24 hours | S3 lifecycle rule |
| Transcribe output | 1 hour | S3 lifecycle rule |
| Conversation messages | Indefinite (user-controlled) | Manual delete |
| Nutrition logs | Indefinite | Manual delete |
| Recipes | Indefinite | Manual delete |
