<div align="center">

# 🥗 AI Health & Nutrition Assistant

### Intelligent. Personalised. Built for 1.4 Billion People.

*The AI health companion that actually understands Indian food, Indian languages, and Indian lives.*

[![Flutter](https://img.shields.io/badge/Flutter-Android-02569B?style=flat&logo=flutter&logoColor=white)](https://flutter.dev)
[![AWS SAM](https://img.shields.io/badge/IaC-AWS_SAM-FF9900?style=flat&logo=amazonaws&logoColor=white)](https://aws.amazon.com/serverless/sam)
[![Bedrock](https://img.shields.io/badge/AI-Amazon_Bedrock_Nova-7B2D8B?style=flat&logo=amazonaws&logoColor=white)](https://aws.amazon.com/bedrock)
[![Lambda](https://img.shields.io/badge/Compute-13_Lambda_Functions-FF9900?style=flat&logo=awslambda&logoColor=white)](https://aws.amazon.com/lambda)
[![DynamoDB](https://img.shields.io/badge/Database-12_DynamoDB_Tables-4053D6?style=flat&logo=amazondynamodb&logoColor=white)](https://aws.amazon.com/dynamodb)
[![Version](https://img.shields.io/badge/Version-1.0.92%2B97-blue?style=flat)](.)
[![Status](https://img.shields.io/badge/Status-🟢_Live_in_Production-brightgreen?style=flat)](.)
[![Region](https://img.shields.io/badge/AWS_Region-ap--south--1_Mumbai-orange?style=flat)](.)

</div>

---

## The Opportunity

**India has a nutrition crisis hiding in plain sight.**

- 74% of Indians are nutritionally deficient in at least one macro or micronutrient
- Diabetes, obesity, and lifestyle diseases are at epidemic levels — and accelerating
- The global health app market is worth $60B+ — yet almost none of it serves Indian dietary reality
- Existing apps use Western food databases, English-only interfaces, and generic AI with zero personal context

**This product exists to change that.**

A production-grade, AI-powered health and nutrition assistant — built ground-up for Indian households, Indian cuisine, and Indian languages. Not a wrapper around a generic chatbot. A deeply contextual health intelligence platform that knows what's in your kitchen, what you've eaten this week, and what your body actually needs.


---

## What Sets This Apart

| | Generic Health Apps | This Product |
|--|-------------------|-------------|
| Food intelligence | Western calorie database | Indian cuisine — veg · egg · non-veg · vegan |
| AI awareness | Stateless chatbot | Knows your pantry, meals, nutrition gaps, goals |
| Language | English only | 6 Indian languages with voice input |
| Auth model | Email + password | Phone + OTP — no email needed |
| Payment | International only | Razorpay — UPI · cards · netbanking |
| AI engine | Generic LLM | Amazon Bedrock Nova — context-built per user per message |
| Architecture | Monolith or basic API | 13 serverless Lambda functions, fully event-driven |
| Cost model | Fixed servers | Pay-per-use, Graviton2, 78%+ gross margin |

---

## Core Product Features

### 🧠 Context-Aware AI Health Coach
This is not a chatbot. This is a health intelligence engine.

Before every single AI response, the system dynamically assembles your complete health context:

- Your dietary preference and food restrictions
- Your health goals (weight loss / muscle gain / maintenance)
- Your current pantry inventory — what ingredients you actually have
- Items expiring in the next 7 days — so nothing goes to waste
- Today's nutrition intake — calories, protein, carbs, fat consumed vs target
- Your recent meal history — what you've eaten over the past 3 days
- Your full conversation history — so it remembers what you discussed

The result: when you ask *"what should I eat for dinner?"* — the AI doesn't give a generic answer. It looks at your shelf, your nutrition gap for the day, your health goal, and your dietary preference — and gives you a specific, actionable recommendation that fits your actual life.

**Powered by:** Amazon Bedrock Nova Chat · Dynamic system prompt per user · DynamoDB context pipeline

---

### 🍳 AI Recipe Generation
Generate personalised Indian recipes from your actual pantry.

- Amazon Bedrock Nova Text generates step-by-step recipes
- Recipes tailored to what you have, your dietary type, and your health goal
- Pre-seeded recipe database: hundreds of vegetarian, egg, non-veg, and vegan Indian dishes
- Save, browse, and reuse recipes across sessions

---

### 📸 Food Photo Recognition
Point your camera at a meal — AI does the rest.

- Amazon Bedrock Nova Vision analyses the photo
- Identifies Indian food items, estimates serving size, calories, and macros
- Bulk review screen for multi-item meals
- One tap to log directly into your nutrition tracker

---

### 📅 Intelligent Meal Planner
Plan your week. Track your nutrition. Hit your goals.

- Daily and weekly meal planning with drag-and-drop simplicity
- Indian calorie database — not generic Western data
- Real-time nutrition summary per day (calories, protein, carbs, fat)
- Meal history feeds back into the AI context for smarter recommendations

---

### 🥫 Smart Pantry Manager (MyShelf)
Know what you have. Never waste food. Always know what to cook.

- Track ingredients with quantities and expiry dates
- AI proactively surfaces expiring items in every chat session
- Shelf inventory is a live input to recipe generation and AI recommendations
- Voice commands to add and check items *(in progress)*

---

### 📊 Nutrition Intelligence
Understand what your body is actually getting.

- Manual food logging with Indian calorie lookup
- Photo-based food identification → auto-populated log entry
- Daily macro breakdown (calories, protein, carbs, fat)
- Health insights screen with trends and gap analysis
- Nutrition data feeds directly into AI context for personalised advice

---

### 🔔 Wellness Reminders & Activity Tracking
- Scheduled wellness notifications (Android local notifications)
- Step counter integration via device pedometer
- Health insights screen with activity and nutrition trends

---

### 🎤 Multilingual Voice Assistant *(In Progress)*
The full voice pipeline is built and being integrated.

- AWS Transcribe for speech-to-text — 6 Indian languages, auto-detected
- AWS Polly for text-to-speech — Standard and Neural voices
- Local audio caching service (70% cost reduction on TTS)
- Language selector: Hindi · English · Tamil · Telugu · Malayalam · Kannada
- Covers ~892 million people — 65% of India's population

---

## Technology Stack

### 📱 Mobile App — Flutter (Android)

| Layer | Technology | Why |
|-------|-----------|-----|
| Framework | Flutter (Dart) | Single codebase, native performance, rich UI |
| State Management | Provider | Lightweight, predictable, testable |
| Authentication | AWS Amplify + Amazon Cognito | Managed auth, JWT tokens, zero password storage |
| Payments | Razorpay Flutter SDK | India-first, UPI + cards + netbanking |
| Voice Input | AWS Transcribe | Best-in-class Indian language STT |
| Voice Output | AWS Polly | Natural TTS, Standard + Neural voices |
| Notifications | flutter_local_notifications | Scheduled wellness reminders |
| Local Storage | shared_preferences (user-scoped) | Cross-user data isolation |
| Image | image_picker + flutter_image_compress | Camera + gallery, optimised upload |
| Activity | pedometer | Step counter, no third-party dependency |

### ☁️ Backend — AWS Serverless

| Layer | Technology | Why |
|-------|-----------|-----|
| Runtime | Node.js 20.x · arm64 Graviton2 | 20% cheaper than x86, same performance |
| IaC | AWS SAM (CloudFormation) | Full infrastructure as code, multi-env |
| API | Amazon API Gateway (REST) | Managed, scalable, Cognito-integrated |
| Auth | Amazon Cognito + Custom Challenge | Phone OTP, no email, no passwords |
| OTP Delivery | Fast2SMS | Indian SMS gateway, high deliverability |
| AI — Conversation | Amazon Bedrock Nova Chat | Context-aware, low latency, cost-efficient |
| AI — Generation | Amazon Bedrock Nova Text | Recipe and content generation |
| AI — Vision | Amazon Bedrock Nova Vision | Food photo recognition |
| Voice STT | AWS Transcribe | 6 Indian languages, batch mode |
| Voice TTS | AWS Polly | Neural voices, audio caching |
| Database | Amazon DynamoDB (×12, on-demand) | Zero idle cost, auto-scales |
| Storage | Amazon S3 | Food photos, voice cache, Glacier lifecycle |
| Secrets | AWS Secrets Manager | Payment credentials, never in code |
| Monitoring | CloudWatch · X-Ray | Logs, metrics, distributed tracing |
| CDN | Amazon CloudFront | Low-latency asset delivery |
| CI/CD | GitHub Actions → SAM Deploy | Zero-downtime Lambda deployments |


---

## System Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                      Flutter Android App                              │
│  AI Chat · Recipes · Meal Planner · Pantry · Nutrition · Voice       │
└──────────────────────────┬───────────────────────────────────────────┘
                           │  HTTPS + Cognito JWT (every request)
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                 Amazon API Gateway  (REST API)                        │
│          Cognito JWT Authorizer · Request Validation · CORS           │
└───┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬────────────────┘
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
        users · recipes             food photos
        meals · shelf               voice audio cache
        nutrition · conversations   (Glacier lifecycle)
        subscriptions · alarms
               │
     ┌─────────┴──────────┐
     ▼                    ▼
Amazon Bedrock        AWS Transcribe
Nova Chat             + AWS Polly
Nova Text             Voice pipeline
Nova Vision           6 Indian languages
(context-aware AI)    STT + TTS
```

### 13 Lambda Functions — Each Purpose-Built

| Function | Responsibility | Memory | Timeout |
|----------|---------------|:------:|:-------:|
| auth-handler | Phone OTP via Fast2SMS + Cognito custom challenge | 256MB | 10s |
| user-handler | User profile CRUD + health data | 256MB | 10s |
| conversation-handler | AI chat session lifecycle | 256MB | 10s |
| message-handler | Message processing + conversation history | 256MB | 10s |
| recipe-handler | Recipe CRUD + Bedrock AI generation | 256MB | 10s |
| meal-plan-handler | Meal planning + nutrition calculation | 256MB | 10s |
| nutrition-log-handler | Food logging + calorie lookup | 256MB | 10s |
| shelf-item-handler | Pantry inventory + expiry tracking | 256MB | 10s |
| voice-handler | Full STT → AI → TTS pipeline | 512MB | 30s |
| bedrock-ai-handler | Bedrock orchestration + context assembly | 512MB | 30s |
| subscription-handler | Razorpay billing + webhook processing | 256MB | 15s |
| alarm-handler | Wellness reminders + scheduling | 128MB | 10s |
| admin-handler | Operations APIs *(planned)* | 256MB | 10s |

### 12 DynamoDB Tables — Zero Idle Cost

| Table | What It Stores |
|-------|---------------|
| users | Profiles, health goals, dietary preference, subscription tier |
| auth-tokens | OTP tokens with 5-min TTL (auto-deleted by DynamoDB) |
| conversations | AI chat sessions |
| messages | Full conversation history (context window for AI) |
| recipes | Recipe library — user-created + AI-generated |
| meal-plans | Daily and weekly meal plans |
| nutrition-logs | Food logs with calorie and macro data |
| shelf-items | Pantry inventory with expiry dates |
| subscriptions | Billing records and subscription lifecycle |
| alarms | Wellness reminder schedules |
| consultants | Legacy — scheduled for removal |
| bookings | Legacy — scheduled for removal |

---

## The AI Context Pipeline — What Makes This Genuinely Intelligent

Most "AI health apps" send a message to an LLM and get a generic response back.
This product does something fundamentally different.

```
User sends: "What should I have for dinner?"
                    ↓
bedrock-ai-handler assembles context in parallel:

  DynamoDB read 1 → User profile
                    dietary: vegetarian
                    goal: weight loss
                    age: 34, BMI: 27.2

  DynamoDB read 2 → Shelf items
                    paneer (500g), tomatoes (expiring tomorrow),
                    spinach, onions, cumin, coriander

  DynamoDB read 3 → Today's nutrition
                    calories: 1,180 / 1,600 target
                    protein: 28g / 55g target  ← gap detected
                    carbs: 180g / 200g target

  DynamoDB read 4 → Recent meals (3 days)
                    breakfast: poha, lunch: dal rice

  DynamoDB read 5 → Last 10 messages
                    (conversation continuity)
                    ↓
Dynamic system prompt constructed with all context
                    ↓
Amazon Bedrock Nova Chat invoked
                    ↓
Response: "You have paneer and spinach that need using —
           palak paneer would give you ~28g protein,
           closing your protein gap for today.
           Tomatoes expire tomorrow so use them in the gravy.
           That puts you at 1,580 cal — right on target."
```

This is not a chatbot. This is a health intelligence engine.

---

## Engineering Depth

### Phone-Only Authentication (No Passwords, No Email)
Custom Cognito `CUSTOM_AUTH` flow with three trigger Lambdas:
- `cognito-define-auth-challenge` — initiates the OTP challenge
- `cognito-create-auth-challenge` — generates OTP, delivers via Fast2SMS, stores with 5-min TTL
- `cognito-verify-auth-challenge` — validates input, marks token used

Result: zero password storage, zero email dependency, single-use OTP, automatic expiry.

### Voice Pipeline (AWS Transcribe + Polly)
```
Flutter AudioRecorder → m4a audio → S3 upload
        ↓
voice-handler Lambda
        ↓
AWS Transcribe (batch) — auto-detects language from 6 options
        ↓
bedrock-ai-handler — full context assembly → Bedrock Nova Chat
        ↓
AWS Polly TTS → MP3 audio
        ↓
S3 audio cache (keyed by response hash, 24h TTL)
        ↓
Flutter streams audio via pre-signed URL
```

### Cost-Optimised Infrastructure
Every architectural decision was evaluated against cost impact:

| Decision | Saving |
|----------|--------|
| Graviton2 (arm64) on all 13 Lambdas | 20% compute reduction |
| On-demand DynamoDB (12 tables) | Zero idle cost — pay per request |
| S3 Glacier lifecycle after 90 days | 90% storage cost reduction |
| API Gateway response caching | 60% fewer Lambda invocations |
| Voice audio caching (24h TTL) | 70% TTS cost reduction |
| Batch transcription vs streaming | 50% cheaper STT |
| Per-tier AI usage limits | Prevents runaway Bedrock costs |

**Estimated gross margin at scale: 78%+**

### Security by Design
- Every API endpoint protected by Cognito JWT authorizer
- IAM least-privilege: each Lambda has its own role with only the permissions it needs
- All DynamoDB queries scoped by `user_id` — cross-user data access is architecturally impossible
- Flutter uses user-scoped local storage — no data bleeds between accounts on shared devices
- Payment credentials live exclusively in AWS Secrets Manager
- S3 bucket is private — all access via pre-signed URLs with 5-minute expiry
- OTP tokens auto-deleted by DynamoDB TTL — no manual cleanup needed

### CI/CD — Zero-Downtime Deployments
```
git push → GitHub Actions
         → sam build (Lambda packages)
         → sam deploy (CloudFormation changeset)
         → Lambda updated with zero downtime
         → Flutter APK built + signed (Android keystore)
```
Three environments: `dev` · `staging` · `prod` — all parameterised via SAM config.

---

## App Screens (26 Screens, Production)

| Screen | Purpose |
|--------|---------|
| login | Phone number entry |
| otp | OTP verification |
| register | New user onboarding + health profile setup |
| home | Dashboard — calories, shelf count, meals planned |
| chat (enhanced) | Context-aware AI health coach |
| voice_chat | Voice assistant interface |
| recipes | Recipe library |
| generate_recipe | AI recipe generation from shelf |
| recipe_detail | Full recipe with nutrition info |
| planner | Weekly meal planner |
| add_meal | Add meal to plan |
| food_log | Nutrition log |
| food_capture | Photo-based food identification |
| bulk_review | Review AI-identified food items |
| select_recipe_for_log | Log a saved recipe as a meal |
| shelf | Pantry manager |
| add_shelf_item | Add item to pantry |
| health_insights | Nutrition trends and analytics |
| wellness_reminder | Reminder setup |
| wellness_reminders | Manage all active reminders |
| subscription | Subscription management |
| subscription_selection | Tier selection and upgrade |
| profile | User health profile |
| settings | App settings |
| navigation | Bottom navigation shell |
| update_required | Force update gate for breaking API changes |

---

## Roadmap

| | Feature | Status |
|--|---------|:------:|
| ✅ | Context-aware AI health coach (Bedrock Nova Chat) | **Live** |
| ✅ | AI recipe generation (Bedrock Nova Text) | **Live** |
| ✅ | Food photo recognition (Bedrock Nova Vision) | **Live** |
| ✅ | Meal planner + nutrition logging | **Live** |
| ✅ | Smart pantry manager (MyShelf) | **Live** |
| ✅ | Phone OTP auth (Cognito + Fast2SMS) | **Live** |
| ✅ | Razorpay recurring billing | **Live** |
| ✅ | Wellness reminders + step counter | **Live** |
| ✅ | 90+ production build iterations | **Live** |
| 🔄 | Multilingual voice assistant (6 Indian languages) | In Progress |
| 📋 | Admin control panel | Planned |
| 📋 | Family health dashboard (up to 5 members) | Planned |
| 📋 | Google Play Store launch | Planned |

---

## Deep Dives

| Document | What's Inside |
|----------|--------------|
| [Architecture](./architecture.md) | System layers, API design, Cognito custom auth, Lambda inventory, scalability path |
| [AI Intelligence](./ai.md) | Bedrock Nova Chat/Text/Vision, context assembly pipeline, voice STT+TTS, prompt engineering, cost optimisation |
| [Mobile App](./mobile.md) | Flutter architecture, 26 screens, service layer, voice UI components, build & release |
| [Data Architecture](./data.md) | 12 DynamoDB schemas, S3 structure, context query pattern, data lifecycle |
| [Security](./security.md) | IAM per-Lambda, data isolation, secrets management, auth design |
| [Infrastructure & Cost](./infrastructure.md) | SAM IaC, cost at scale, CI/CD pipeline, monitoring, multi-environment |
