<div align="center">

# 🥗 Cojaisy
### AI Health & Nutrition Assistant

*Voice-first. Multilingual. Built for India.*

[![Flutter](https://img.shields.io/badge/Flutter-Android-02569B?style=flat&logo=flutter&logoColor=white)](https://flutter.dev)
[![AWS SAM](https://img.shields.io/badge/AWS_SAM-Serverless-FF9900?style=flat&logo=amazonaws&logoColor=white)](https://aws.amazon.com/serverless/sam)
[![Amazon Bedrock](https://img.shields.io/badge/Amazon_Bedrock-Nova_Chat_·_Vision_·_Text-7B2D8B?style=flat&logo=amazonaws&logoColor=white)](https://aws.amazon.com/bedrock)
[![DynamoDB](https://img.shields.io/badge/DynamoDB-12_Tables-4053D6?style=flat&logo=amazondynamodb&logoColor=white)](https://aws.amazon.com/dynamodb)
[![Version](https://img.shields.io/badge/Version-1.0.92%2B97-blue?style=flat)](.)
[![Status](https://img.shields.io/badge/Status-🟢_Live_in_Production-brightgreen?style=flat)](.)
[![Region](https://img.shields.io/badge/Region-ap--south--1_Mumbai-orange?style=flat)](.)

</div>

---

## The Problem

Most nutrition and health apps are built for Western diets and English speakers. Indian users — with diverse regional cuisines, 6+ languages, and family-centric eating habits — are completely underserved by generic calorie trackers.

**Cojaisy is built for India, not adapted for it.**

- Indian food calorie database (not generic Western data)
- 6 regional languages with natural voice responses
- Phone-based auth — no email required (matches Indian user behaviour)
- Indian payment gateway (Razorpay) with UPI + card + netbanking
- Recipes seeded with vegetarian, egg, non-veg, and vegan Indian cuisine

---

## What It Does

| Feature | Description |
|---------|-------------|
| 🎤 **Voice Assistant** | Ask health questions, add shelf items, generate recipes — all by voice in 6 Indian languages |
| 🤖 **AI Chat** | Context-aware health & nutrition Q&A powered by Amazon Bedrock Nova |
| 📸 **Food Recognition** | Photograph a meal — AI identifies the food and estimates calories (Bedrock Vision) |
| 🍳 **Recipe Engine** | AI generates personalised Indian recipes from your pantry inventory |
| 📅 **Meal Planner** | Daily and weekly meal planning with nutrition tracking |
| 🥫 **MyShelf** | Smart pantry manager — track ingredients, get recipe suggestions |
| 📊 **Nutrition Logging** | Manual + photo-based food logging with Indian calorie database |
| 🔔 **Wellness Reminders** | Scheduled notifications + step counter integration |
| 💳 **Subscriptions** | Free → Wellness (₹99/mo) → FamilyCare (₹499/mo) via Razorpay recurring billing |
| 🖥️ **Admin Dashboard** | React web app on S3 + CloudFront — user management, analytics, infrastructure health |

---

## Technology Stack

### 📱 Mobile App (Flutter / Android)

| Layer | Technology |
|-------|-----------|
| Framework | Flutter (Dart) |
| State Management | Provider |
| Authentication | AWS Amplify + Amazon Cognito |
| Payments | Razorpay Flutter SDK (recurring subscriptions) |
| Voice Input | AWS Transcribe (Speech-to-Text) |
| Voice Output | AWS Polly (Text-to-Speech) |
| Notifications | flutter_local_notifications |
| Local Storage | shared_preferences (user-scoped isolation) |
| Image Handling | image_picker + flutter_image_compress |
| Step Counter | pedometer |

### ☁️ Backend (AWS Serverless)

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js 20.x · arm64 Graviton2 |
| Infrastructure as Code | AWS SAM (CloudFormation) |
| API | Amazon API Gateway (REST) + Cognito JWT Authorizer |
| Compute | 13 AWS Lambda functions |
| Database | 12 Amazon DynamoDB tables (on-demand) |
| Authentication | Amazon Cognito User Pool (phone + OTP) |
| OTP Delivery | Fast2SMS (Indian SMS gateway) |
| AI Engine | Amazon Bedrock — Nova Chat · Nova Text · Nova Vision |
| Voice STT | AWS Transcribe (6 Indian languages, auto-detect) |
| Voice TTS | AWS Polly (Standard + Neural voices, tier-based) |
| Storage | Amazon S3 (food photos + voice audio cache) |
| Secrets | AWS Secrets Manager (Razorpay credentials) |
| Monitoring | CloudWatch Logs · Metrics · X-Ray tracing |
| CDN | Amazon CloudFront (admin dashboard) |
| CI/CD | GitHub Actions → SAM Deploy |

### 🖥️ Admin Dashboard (React / Web)

| Layer | Technology |
|-------|-----------|
| Framework | React |
| Hosting | Amazon S3 + CloudFront |
| Authentication | Amazon Cognito (admin-only user pool, MFA) |
| Backend | admin-handler Lambda + DynamoDB |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    Flutter Android App                            │
│   Auth · Voice · Chat · Recipes · Shelf · Meals · Subscriptions  │
└────────────────────────┬─────────────────────────────────────────┘
                         │  HTTPS + Cognito JWT
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│             Amazon API Gateway  (REST API)                        │
│             Cognito Authorizer · Request Validation · CORS        │
└──┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬─────────────┘
   │      │      │      │      │      │      │      │
   ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼
 auth   user  recipe  meal  shelf  voice bedrock  sub
  λ      λ     λ      plan   item    λ    ai-λ    λ
                       λ      λ
   │      │      │      │      │      │      │      │
   └──────┴──────┴──────┴──────┴──────┴──────┴──────┘
                         │
             ┌───────────┴────────────┐
             ▼                        ▼
      DynamoDB (12 tables)        Amazon S3
      users · recipes · meals     food photos
      shelf · subscriptions       voice cache
      conversations · logs
             │
   ┌─────────┴──────────┐
   ▼                    ▼
Amazon Bedrock      AWS Transcribe
Nova Chat/Text/     + AWS Polly
Vision              (Voice I/O,
(AI · recipes ·      6 languages)
 food recognition)
```

### Lambda Functions (13)

| Function | Purpose | Memory | Timeout |
|----------|---------|:------:|:-------:|
| auth-handler | Phone OTP auth via Fast2SMS + Cognito custom challenge | 256MB | 10s |
| user-handler | User profile CRUD | 256MB | 10s |
| conversation-handler | AI chat session management | 256MB | 10s |
| message-handler | Chat message processing + history | 256MB | 10s |
| recipe-handler | Recipe CRUD + Bedrock AI generation | 256MB | 10s |
| meal-plan-handler | Meal planning + nutrition tracking | 256MB | 10s |
| nutrition-log-handler | Food logging + calorie lookup | 256MB | 10s |
| shelf-item-handler | Pantry/shelf management | 256MB | 10s |
| voice-handler | STT + AI + TTS full pipeline | 512MB | 30s |
| bedrock-ai-handler | Bedrock orchestration (chat + vision) | 512MB | 30s |
| subscription-handler | Razorpay subscriptions + webhooks | 256MB | 15s |
| alarm-handler | Wellness reminders + scheduling | 128MB | 10s |
| admin-handler | Admin dashboard APIs + analytics | 256MB | 10s |

### DynamoDB Tables (12)

| Table | Purpose |
|-------|---------|
| kiro-users | Profiles, subscription tier, health goals, usage counters |
| kiro-auth-tokens | OTP tokens with 5-min TTL (auto-deleted) |
| kiro-conversations | AI chat sessions |
| kiro-messages | Chat message history (context window) |
| kiro-recipes | Recipe library (user-created + AI-generated) |
| kiro-meal-plans | Daily/weekly meal plans |
| kiro-nutrition-logs | Food logs with calorie + macro data |
| kiro-shelf-items | Pantry inventory |
| kiro-subscriptions | Razorpay subscription records |
| kiro-alarms | Wellness reminder schedules |
| kiro-consultants | Consultant profiles (legacy) |
| kiro-bookings | Booking records (legacy) |

---

## Authentication Flow

Phone-only OTP — no passwords, no email required.

```
User enters phone number
        ↓
Cognito CUSTOM_AUTH flow initiated
        ↓
cognito-create-auth-challenge Lambda
  → generates 6-digit OTP
  → calls Fast2SMS API (Indian SMS delivery)
  → stores OTP in DynamoDB (5-min TTL)
        ↓
User enters OTP
        ↓
cognito-verify-auth-challenge Lambda validates
        ↓
Cognito issues: id_token + access_token + refresh_token (JWT)
        ↓
Flutter stores tokens in user-scoped local storage
        ↓
All API calls: Authorization: Bearer <access_token>
```

---

## Voice AI Pipeline

```
Flutter AudioRecorder → m4a audio
        ↓
Upload to S3: voice-input/{user_id}/{timestamp}.m4a
        ↓
voice-handler Lambda
        ↓
AWS Transcribe (batch) — auto-detects language
  Supported: en-IN · hi-IN · ta-IN · te-IN · ml-IN · kn-IN
        ↓
bedrock-ai-handler — Amazon Bedrock Nova Chat
  Context: user's shelf + meal history + health goals
  Response: max 200 words (voice-optimised)
  Intent: add_to_shelf | generate_recipe | meal_plan | general
        ↓
AWS Polly TTS
  Free tier  → Standard voice
  Paid tiers → Neural voice (more natural)
        ↓
Audio cached in S3 (24h TTL, keyed by response hash)
        ↓
Flutter plays audio via pre-signed URL
```

**Cost optimisations in the voice pipeline:**

| Optimisation | Saving |
|-------------|--------|
| Audio response caching (24h) | ~70% TTS cost reduction |
| Response length cap (200 words) | ~60% Polly cost reduction |
| Batch transcription (not streaming) | ~50% Transcribe cost reduction |
| Tier-based voice quality | Standard for Free, Neural for paid |

---

## Subscription Tiers

| | Free | Wellness ₹99/mo | FamilyCare ₹499/mo |
|--|:----:|:---------------:|:-----------------:|
| Voice queries/month | 2 | 100 | 500 (shared) |
| Food photo recognition | 2 | 5 | 50 (shared) |
| AI recipe generation | 2 | 20 | 100 (shared) |
| Manual logging | ✅ Unlimited | ✅ Unlimited | ✅ Unlimited |
| Meal planning | Basic | Full | Full |
| Voice quality | Standard | Neural | Neural |
| Family members | 1 | 1 | Up to 5 |
| Family dashboard | ✗ | ✗ | ✅ |
| **Cost to serve** | ~₹8 | ~₹41 | ~₹267 |
| **Gross margin** | — | **₹58** | **₹232** |

Payments via **Razorpay recurring subscriptions** — UPI, cards, netbanking.
Webhook-driven lifecycle: `subscription.charged` → `subscription.cancelled` → `subscription.paused`.

---

## Cost Engineering

| Optimisation | Impact |
|-------------|--------|
| Graviton2 (arm64) Lambda | 20% compute cost reduction |
| On-demand DynamoDB | Zero idle cost, auto-scales |
| S3 Glacier lifecycle (90 days) | 90% storage cost reduction for old photos |
| API Gateway response caching | 60% reduction in Lambda invocations |
| CloudWatch log retention (7d dev / 30d prod) | Bounded log storage cost |
| Voice audio caching | 70% TTS cost reduction |
| Usage limits per tier | Prevents runaway AI costs |

**Estimated gross margin at 1,000 users (mixed tiers): ~78%**

---

## Security Highlights

- All endpoints protected by Cognito JWT authorizer
- Phone-only auth — zero password storage
- OTP tokens: DynamoDB TTL auto-deletes after 5 minutes
- Razorpay credentials: AWS Secrets Manager (never in code or env vars)
- S3: private bucket, pre-signed URLs (5-min expiry)
- IAM: least-privilege role per Lambda function
- Data isolation: all DynamoDB queries scoped by `user_id`
- Flutter: user-scoped local storage (prevents cross-user data leakage)
- Admin: separate Cognito user pool with MFA

---

## CI/CD

```
git push → GitHub Actions
        ↓
sam build  (Node.js Lambda packages)
        ↓
sam deploy (CloudFormation stack update)
        ↓
Zero-downtime Lambda updates
```

Multi-environment: `dev` · `staging` · `prod` — all via SAM config parameters.

---

## App Screens (20)

`login` · `otp` · `register` · `home` · `chat` · `voice_chat` · `recipes` · `generate_recipe` · `recipe_detail` · `planner` · `add_meal` · `food_log` · `food_capture` · `shelf` · `health_insights` · `wellness_reminder` · `subscription` · `subscription_selection` · `profile` · `settings`

---

## Roadmap

| Phase | Feature | Status |
|-------|---------|:------:|
| Phase 1 | Admin Control Panel | ✅ Complete |
| Phase 2 | Usage Tracking & Limits | ✅ Complete |
| Phase 3 | Voice Action Execution (auto-execute intents) | 🔄 In Progress |
| Phase 4 | Family Health Dashboard | 📋 Planned |
| Phase 5 | "Hey Cojaisy" Wake Word | 📋 Planned |
| Phase 6 | Google Play Store Launch | 📋 Planned |

---

## Deep Dives

| Document | What's Inside |
|----------|--------------|
| [Architecture](./architecture.md) | System layers, API design, data model, Cognito custom auth, failure handling |
| [AI & Voice Pipeline](./ai-voice.md) | Bedrock prompt engineering, voice pipeline, intent detection, food recognition |
| [Subscriptions](./subscriptions.md) | Razorpay recurring billing, webhook lifecycle, usage tracking, admin analytics |
| [Security](./security.md) | IAM per-Lambda table, data isolation, secrets management, known trade-offs |
| [Infrastructure & Cost](./infrastructure.md) | SAM IaC, cost estimates, CI/CD, monitoring, multi-environment setup |
