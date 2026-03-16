<div align="center">

# 🥗 Cojaisy
### AI Health & Nutrition Assistant

*Context-aware. Personalised. Built for India.*

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

Most nutrition and health apps are built for Western diets and English speakers.
Indian users — with diverse regional cuisines, 6+ languages, and family-centric eating habits — are completely underserved by generic calorie trackers.

**Cojaisy is built for India, not adapted for it.**

---

## What Makes It Different

| | Generic Health Apps | Cojaisy |
|--|-------------------|---------|
| Food database | Western foods | Indian cuisine (veg · egg · non-veg · vegan) |
| AI awareness | None | Knows your shelf, meals, goals, dietary preference |
| Language | English only | 6 Indian languages (voice input) |
| Auth | Email + password | Phone + OTP only (no email needed) |
| Payment | International cards | Razorpay — UPI · cards · netbanking |
| AI engine | Generic chatbot | Amazon Bedrock Nova — context-aware per user |


---

## Core Features (Live in Production)

### 🤖 Context-Aware AI Chat
The centrepiece of the app. Not a generic chatbot — an AI health coach that knows everything about you.

Before every response, the AI loads your full context:
- Your dietary preference (vegetarian / egg / non-veg / vegan)
- Your health goals (weight loss / muscle gain / maintenance)
- Your current shelf inventory (what ingredients you have)
- Your recent meal history (what you've eaten this week)
- Your nutrition logs (calories, protein, carbs, fat gaps)
- Items expiring soon in your pantry

This means when you ask *"what should I eat for dinner?"* — it doesn't give a generic answer. It looks at what's in your shelf, what you've already eaten today, and your health goal, then suggests something specific and actionable.

**Powered by:** Amazon Bedrock Nova Chat · Conversation history (DynamoDB) · Dynamic system prompt per user

---

### 🍳 AI Recipe Engine
Generate personalised Indian recipes from your actual pantry inventory.

- Bedrock Nova Text generates step-by-step recipes
- Tailored to dietary preference and health goals
- Pre-seeded recipe database: vegetarian, egg, non-veg, vegan Indian cuisine
- Save, browse, and reuse recipes

---

### 📸 Food Photo Recognition
Photograph a meal — AI identifies the food and estimates calories.

- Bedrock Nova Vision analyses the image
- Returns: food name, serving size, calories, macros
- Bulk review screen for multi-item meals
- Feeds directly into nutrition log

---

### 📅 Meal Planner
Daily and weekly meal planning with full nutrition tracking.

- Plan breakfast, lunch, dinner, snacks
- Indian calorie database (not generic Western data)
- Nutrition summary per day (calories, protein, carbs, fat)
- Bulk review and approval flow

---

### 🥫 MyShelf — Smart Pantry Manager
Track your ingredients. Never waste food. Always know what to cook.

- Add items with quantities and expiry dates
- AI chat proactively warns about items expiring soon
- Shelf inventory feeds into recipe generation and AI chat context

---

### 📊 Nutrition Logging
Manual + photo-based food logging.

- Search Indian food items with calorie data
- Photo capture → AI identification → auto-populated log entry
- Daily nutrition summary and health insights screen

---

### 🔔 Wellness Reminders
- Scheduled wellness notifications (Android local notifications)
- Step counter integration (pedometer)
- Health insights screen with trends

---

### 🎤 Voice Assistant *(In Progress)*
Voice input + voice response pipeline is built and partially integrated.

- AWS Transcribe for speech-to-text (6 Indian languages)
- AWS Polly for text-to-speech (Standard + Neural voices)
- Audio caching service (70% cost reduction)
- Language selector widget (Hindi, English, Tamil, Telugu, Malayalam, Kannada)
- Full integration with AI chat in progress

---

## Technology Stack

### 📱 Mobile App (Flutter / Android)

| Layer | Technology |
|-------|-----------|
| Framework | Flutter (Dart) |
| State Management | Provider |
| Authentication | AWS Amplify + Amazon Cognito |
| Payments | Razorpay Flutter SDK |
| Voice Input | AWS Transcribe (Speech-to-Text) |
| Voice Output | AWS Polly (Text-to-Speech) |
| Notifications | flutter_local_notifications |
| Local Storage | shared_preferences (user-scoped isolation) |
| Image | image_picker + flutter_image_compress |
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
| AI — Chat | Amazon Bedrock Nova Chat |
| AI — Recipes | Amazon Bedrock Nova Text |
| AI — Food Recognition | Amazon Bedrock Nova Vision |
| Voice STT | AWS Transcribe (6 Indian languages) |
| Voice TTS | AWS Polly (Standard + Neural voices) |
| Storage | Amazon S3 (food photos + voice audio cache) |
| Secrets | AWS Secrets Manager (Razorpay credentials) |
| Monitoring | CloudWatch Logs · Metrics · X-Ray tracing |
| CDN | Amazon CloudFront |
| CI/CD | GitHub Actions → SAM Deploy |


---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    Flutter Android App                            │
│   Auth · AI Chat · Recipes · Shelf · Meals · Nutrition · Voice   │
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
                         │
             ┌───────────┴────────────┐
             ▼                        ▼
      DynamoDB (12 tables)        Amazon S3
      users · recipes · meals     food photos
      shelf · conversations       voice cache
      nutrition · subscriptions
             │
   ┌─────────┴──────────┐
   ▼                    ▼
Amazon Bedrock      AWS Transcribe
Nova Chat           + AWS Polly
Nova Text           (Voice pipeline,
Nova Vision          6 Indian languages)
```

### Lambda Functions (13)

| Function | Purpose | Memory | Timeout |
|----------|---------|:------:|:-------:|
| auth-handler | Phone OTP via Fast2SMS + Cognito custom challenge | 256MB | 10s |
| user-handler | User profile CRUD | 256MB | 10s |
| conversation-handler | AI chat session management | 256MB | 10s |
| message-handler | Chat message + context window | 256MB | 10s |
| recipe-handler | Recipe CRUD + Bedrock AI generation | 256MB | 10s |
| meal-plan-handler | Meal planning + nutrition tracking | 256MB | 10s |
| nutrition-log-handler | Food logging + calorie lookup | 256MB | 10s |
| shelf-item-handler | Pantry/shelf management | 256MB | 10s |
| voice-handler | STT + AI + TTS full pipeline | 512MB | 30s |
| bedrock-ai-handler | Bedrock orchestration — chat, vision, context | 512MB | 30s |
| subscription-handler | Razorpay subscriptions + webhooks | 256MB | 15s |
| alarm-handler | Wellness reminders + scheduling | 128MB | 10s |
| admin-handler | Admin APIs (planned) | 256MB | 10s |

---

## Authentication Flow

Phone-only OTP — no passwords, no email required.

```
User enters phone number
        ↓
Cognito CUSTOM_AUTH flow
        ↓
cognito-create-auth-challenge Lambda
  → generates 6-digit OTP
  → delivers via Fast2SMS (Indian SMS)
  → stores in DynamoDB with 5-min TTL
        ↓
User enters OTP
        ↓
cognito-verify-auth-challenge Lambda validates
        ↓
Cognito issues JWT tokens (id + access + refresh)
        ↓
All API calls: Authorization: Bearer <token>
```

---

## Context-Aware AI — How It Works

Every chat message triggers a dynamic context build before hitting Bedrock:

```
User sends message
        ↓
bedrock-ai-handler Lambda
        ↓
Parallel DynamoDB reads:
  → User profile (diet, goals, age, BMI)
  → Shelf items (current inventory)
  → Expiring items (next 7 days)
  → Today's nutrition logs (calories, macros)
  → Recent meal plans (last 3 days)
  → Last 10 messages (conversation history)
        ↓
Dynamic system prompt constructed:
  "User is vegetarian, goal: weight loss,
   has tomatoes + paneer expiring tomorrow,
   ate 1,200 cal today (target: 1,800),
   protein gap: 25g remaining..."
        ↓
Amazon Bedrock Nova Chat invoked
        ↓
Personalised, actionable response
```

This is what separates Cojaisy from a generic health chatbot.

---

## Cost Engineering

| Optimisation | Impact |
|-------------|--------|
| Graviton2 (arm64) Lambda | 20% compute cost reduction |
| On-demand DynamoDB | Zero idle cost |
| S3 Glacier lifecycle (90 days) | 90% storage cost reduction for old photos |
| API Gateway response caching | 60% reduction in Lambda invocations |
| Voice audio caching (24h TTL) | 70% TTS cost reduction |
| Batch transcription (not streaming) | 50% cheaper STT |
| Usage limits per subscription tier | Prevents runaway AI costs |

---

## Security

- All endpoints protected by Cognito JWT authorizer
- Phone-only auth — zero password storage
- OTP tokens: DynamoDB TTL auto-deletes after 5 minutes, single-use
- Razorpay credentials: AWS Secrets Manager only (never in code)
- S3: private bucket, pre-signed URLs (5-min expiry)
- IAM: least-privilege execution role per Lambda function
- Data isolation: all DynamoDB queries scoped by `user_id`
- Flutter: user-scoped local storage (prevents cross-user data leakage)

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
        ↓
Flutter APK build → signed with Android keystore
```

Multi-environment: `dev` · `staging` · `prod` via SAM config parameters.

---

## Roadmap

| Phase | Feature | Status |
|-------|---------|:------:|
| ✅ | Context-aware AI chat (Bedrock Nova) | Live |
| ✅ | AI recipe generation + food photo recognition | Live |
| ✅ | Meal planner + nutrition logging | Live |
| ✅ | MyShelf pantry manager | Live |
| ✅ | Phone OTP auth (Cognito + Fast2SMS) | Live |
| ✅ | Razorpay subscription billing | Live |
| ✅ | Wellness reminders + step counter | Live |
| 🔄 | Voice assistant full integration | In Progress |
| 📋 | Admin control panel (web UI) | Planned |
| 📋 | Family health dashboard (up to 5 members) | Planned |
| 📋 | Google Play Store launch | Planned |

---

## Deep Dives

| Document | What's Inside |
|----------|--------------|
| [Architecture](./architecture.md) | System layers, API design, data model, Cognito custom auth |
| [AI & Voice Pipeline](./ai-voice.md) | Bedrock prompt engineering, context building, voice pipeline |
| [Subscriptions](./subscriptions.md) | Razorpay recurring billing, webhook lifecycle, usage tracking |
| [Security](./security.md) | IAM per-Lambda, data isolation, secrets management |
| [Infrastructure & Cost](./infrastructure.md) | SAM IaC, cost estimates, CI/CD, monitoring |
