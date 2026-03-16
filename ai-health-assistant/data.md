# Data Architecture — Deep Dive

> 12 DynamoDB tables. Zero idle cost. Every schema decision made for query efficiency and user isolation.

[![DynamoDB](https://img.shields.io/badge/Database-Amazon_DynamoDB-4053D6?style=flat&logo=amazondynamodb&logoColor=white)](https://aws.amazon.com/dynamodb)
[![S3](https://img.shields.io/badge/Storage-Amazon_S3-FF9900?style=flat&logo=amazonaws&logoColor=white)](https://aws.amazon.com/s3)

---

## What Data Is Collected — And Why

This is a health product. Users share personal information — their weight, health goals, what they eat, what's in their kitchen. That data is the engine of the AI's intelligence. It is also a responsibility.

Every field stored exists for a specific functional reason. Nothing is collected speculatively.

| Data Category | What's Stored | Why It's Needed |
|--------------|--------------|----------------|
| Identity | Phone number, name | Account creation and OTP authentication |
| Health profile | Age, gender, height, weight, BMI, dietary preference, health goals, allergies | Powers personalised AI recommendations — without this, the AI gives generic advice |
| Pantry inventory | Ingredient names, quantities, expiry dates | AI uses this to suggest recipes and flag waste |
| Meal history | What was eaten, when, nutritional values | AI uses 3-day history to spot nutrition gaps and patterns |
| Nutrition logs | Daily food intake, calories, macros | Tracks progress against health goals; feeds AI context |
| AI conversations | Message history (last 10 per session) | Gives the AI memory within a session — so it doesn't repeat itself |
| Food photos | Images uploaded for recognition | Processed by AI Vision, then stored privately under user's path |
| Voice audio | Audio clips for STT processing | Transcribed and discarded within 24 hours — not retained |
| Billing | Subscription tier, payment reference | Manages access to paid features — no card data stored (handled by payment gateway) |
| Wellness reminders | Scheduled notification times | Delivers user-configured health reminders |

**What is NOT collected:**
- No email address
- No passwords (phone OTP only)
- No location tracking or GPS data
- No contacts or device data
- No card numbers or payment credentials (processed entirely by the payment gateway)

---

## User Profile — The Core of Personalisation

The user profile is the foundation of everything the AI does. It is created once during onboarding and updated by the user at any time.

```
Health profile fields:
  dietary_preference  → vegetarian | egg | non-veg | vegan
  health_goals        → weight_loss | muscle_gain | maintenance
  age, gender         → used for calorie target calculation
  height_cm, weight_kg, bmi → body metrics for nutritional targets
  allergies           → exclusion list for recipe generation
  locality            → regional cuisine context (e.g. South Indian vs North Indian)
```

This profile is:
- Stored encrypted at rest in DynamoDB (AWS-managed encryption)
- Accessible only to the authenticated user via their Cognito JWT
- Never shared with third parties
- Deletable by the user at any time (account deletion removes all records)

---

## Design Principles

- **On-demand capacity** — zero provisioned throughput, zero idle cost, auto-scales to any spike
- **User-scoped partition keys** — `user_id` as PK on every table ensures data isolation at the storage layer
- **TTL for ephemeral data** — OTP tokens auto-deleted by DynamoDB, no cleanup Lambda needed
- **GSIs for access patterns** — phone number lookup, conversation history, recipe browsing
- **Minimal item size** — only store what the application needs, no bloated documents

---

## Table Schemas

### Users Table
The central user record — health profile, subscription state, AI usage counters.

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
payment_subscription_id   ← billing reference
voice_count_month         ← AI usage tracking (reset monthly)
photo_count_month
recipe_count_month
usage_reset_date
created_at
updated_at
```

GSI: `phone_number-index` — enables Cognito custom auth challenge to look up user by phone.

---

### Auth Tokens Table
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

### Conversations Table
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

### Messages Table
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

### Recipes Table
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

### Meal Plans Table
Daily meal planning records.

```
plan_id (PK)              ← UUID
user_id                   ← GSI PK
date                      ← YYYY-MM-DD
meal_type                 ← breakfast | lunch | dinner | snack
recipe_id                 ← reference to recipes table
meal_name                 ← display name
nutrition                 ← JSON {calories, protein, carbs, fat}
servings
notes
created_at
```

GSI: `user_id-date-index` — fetch all meals for a user on a specific date.

---

### Nutrition Logs Table
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
photo_reference           ← if source = photo
created_at
```

GSI: `user_id-date-index` — fetch all logs for a user on a date (today's nutrition for AI context).

---

### Shelf Items Table
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

### Subscriptions Table
Billing records and subscription lifecycle.

```
subscription_id (PK)      ← UUID
user_id                   ← GSI PK
payment_subscription_id   ← payment gateway reference
tier                      ← wellness | familycare
status                    ← created | active | cancelled | paused
amount
currency
start_date
end_date
next_billing_date
cancelled_at
created_at
updated_at
```

---

### Alarms Table
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

All objects are stored in a single private bucket, organised by data type and user identity:

```
private-bucket/
├── food-photos/
│   └── {user_id}/          ← user-scoped, no cross-user access
│       └── {timestamp}.jpg ← food recognition photos
├── voice-input/
│   └── {user_id}/          ← user-scoped audio uploads
│       └── {timestamp}.m4a ← voice query audio
├── voice-cache/
│   └── {content_hash}.mp3  ← cached Polly responses (24h TTL)
└── transcripts/
    └── {job_id}.json       ← Transcribe output (auto-deleted after 1h)
```

All objects accessed via pre-signed URLs (5-minute expiry). No public access. No permanent links.

---

## Context Assembly Query Pattern

The AI context pipeline runs these queries in parallel before every chat message:

```javascript
const [user, shelfItems, expiringItems, todayNutrition, recentMeals, messages] =
  await Promise.all([
    // User profile
    dynamodb.get({ Key: { user_id } }),

    // Full shelf inventory
    dynamodb.query({
      KeyConditionExpression: 'user_id = :uid',
      ExpressionAttributeValues: { ':uid': userId },
      Limit: 20
    }),

    // Items expiring in next 7 days
    dynamodb.query({
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
      IndexName: 'user_id-date-index',
      KeyConditionExpression: 'user_id = :uid AND #date = :today',
      ExpressionAttributeValues: { ':uid': userId, ':today': today }
    }),

    // Recent meal history (3 days)
    dynamodb.query({
      IndexName: 'user_id-date-index',
      KeyConditionExpression: 'user_id = :uid AND #date >= :threeDaysAgo',
      ExpressionAttributeValues: { ':uid': userId, ':threeDaysAgo': threeDaysAgo }
    }),

    // Last 10 messages (conversation history)
    dynamodb.query({
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

## Data Protection & Encryption

All user data is protected at every layer — in transit, at rest, and at the access control level.

| Layer | Protection |
|-------|-----------|
| In transit | All API calls over HTTPS — no unencrypted channels |
| At rest — DynamoDB | AWS-managed encryption (AES-256) on all 12 tables |
| At rest — S3 | Server-side encryption on all objects in the private bucket |
| Access control | Every DynamoDB query requires `user_id` — no query can return another user's data |
| File access | All S3 objects accessed via pre-signed URLs with 5-minute expiry — no permanent links |
| Device | All locally cached data prefixed with `user_id` — switching accounts clears previous user's data |
| Credentials | Payment keys and SMS keys stored in AWS Secrets Manager — never in code or config |

---

## Account Deletion & Data Removal

When a user deletes their account, all personal data is removed:

| Data | Action on Deletion |
|------|--------------------|
| User profile (health data, goals, preferences) | Deleted from database |
| Conversation history and AI messages | Deleted from database |
| Nutrition logs | Deleted from database |
| Meal plans | Deleted from database |
| Pantry inventory | Deleted from database |
| User-created recipes | Deleted from database |
| Wellness reminders | Deleted from database |
| Food photos | Deleted from S3 |
| Voice audio | Already auto-deleted within 24h — nothing to remove |
| Cognito account | Deleted from user pool — phone number released |
| Subscription record | Marked cancelled, retained for billing audit trail only |

Pre-seeded recipes (not created by the user) remain in the shared library — they contain no personal data.

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
