# Cojaisy — AI Health & Nutrition Assistant

> An AI-powered lifestyle companion for Indian households — voice-first, multilingual,
> and built entirely on serverless AWS infrastructure.

**Platform:** Android (Flutter)
**Backend:** AWS Serverless (SAM)
**AI Engine:** Amazon Bedrock (Nova Chat, Nova Text, Nova Vision)
**Current Version:** 1.0.92+97
**Region:** ap-south-1 (Mumbai)
**Status:** 🟢 Live in Production

---

## The Problem

Most nutrition and health apps are built for Western diets and English speakers.
Indian users — with diverse regional cuisines, 6+ languages, and family-centric
eating habits — are underserved by generic calorie trackers.

Cojaisy solves this by combining:
- Deep Indian food knowledge (vegetarian, egg, non-veg, vegan recipe databases)
- Multilingual voice AI (6 Indian languages)
- Personalized meal planning and shelf management
- A fully serverless, cost-optimized AWS backend

---

## Product Features

### AI Chat & Voice Assistant
- Natural language health and nutrition Q&A
- Voice input + voice response (AWS Transcribe + AWS Polly)
- 6 Indian languages: English, Hindi, Tamil, Telugu, Malayalam, Kannada
- Context-aware responses using user's shelf, meal history, and health goals
- Intent detection: add to shelf, generate recipe, plan meals — all via voice

### Smart Recipe Engine
- AI-generated recipes using Amazon Bedrock (Nova)
- Recipes tailored to shelf inventory, dietary preference, and health goals
- Pre-seeded recipe database: vegetarian, egg, non-veg, vegan (Indian cuisine)
- Photo-based food recognition using Bedrock Vision (Nova Vision)

### Meal Planner
- Daily and weekly meal planning
- Nutrition tracking per meal (calories, macros)
- Indian calorie database (local food items)
- Bulk review and approval flow

### MyShelf (Pantry Manager)
- Track ingredients and pantry items
- AI-powered shelf-to-recipe suggestions
- Voice commands to add/check items

### Nutrition Logging
- Manual food logging with calorie lookup
- Photo capture and AI-based food identification
- Daily nutrition summary and health insights

### Wellness Reminders
- Scheduled wellness notifications (Android local notifications)
- Step counter integration (pedometer)
- Health insights screen

### Subscription & Monetisation
- Free tier with usage limits
- Wellness tier: ₹99/month (Razorpay recurring subscriptions)
- FamilyCare tier: ₹499/month (up to 5 family members, shared limits)
- Usage tracking per user per month (voice, photo, recipe counts)
- Upgrade prompts with tier comparison

### Admin Control Panel
- React web app hosted on S3 + CloudFront
- User management, subscription analytics, AI usage stats
- Infrastructure health monitoring (Lambda, DynamoDB, API Gateway, Bedrock)
- Audit logs and cost analytics

---

## Technology Stack

### Mobile (Frontend)
| Layer | Technology |
|-------|-----------|
| Framework | Flutter (Dart) — Android |
| State Management | Provider |
| Auth | AWS Amplify + Amazon Cognito |
| Payments | Razorpay Flutter SDK |
| Voice | AWS Transcribe (STT) + AWS Polly (TTS) |
| Notifications | flutter_local_notifications |
| Storage | shared_preferences (user-scoped) |
| Image | image_picker + flutter_image_compress |
| Step Counter | pedometer |

### Backend (AWS Serverless)
| Layer | Technology |
|-------|-----------|
| Runtime | Node.js 20.x on arm64 (Graviton2) |
| IaC | AWS SAM (CloudFormation) |
| API | Amazon API Gateway (REST) with Cognito authorizer |
| Compute | 13 AWS Lambda functions |
| Database | 12 Amazon DynamoDB tables (on-demand) |
| Auth | Amazon Cognito User Pool (phone + OTP) |
| OTP Delivery | Fast2SMS (Indian SMS gateway) |
| AI | Amazon Bedrock — Nova Chat, Nova Text, Nova Vision |
| Voice STT | AWS Transcribe (6 Indian languages) |
| Voice TTS | AWS Polly (Standard + Neural voices) |
| Storage | Amazon S3 (food photos + admin dashboard) |
| Secrets | AWS Secrets Manager (Razorpay keys) |
| Monitoring | Amazon CloudWatch Logs, Metrics, X-Ray |
| CDN | Amazon CloudFront (admin dashboard) |
| CI/CD | GitHub Actions |

### Admin Dashboard (Web)
| Layer | Technology |
|-------|-----------|
| Framework | React |
| Hosting | Amazon S3 + CloudFront |
| Auth | Amazon Cognito (admin-only user pool) |
| Data | DynamoDB via admin Lambda |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Flutter Android App                          │
│  Auth │ Voice │ Chat │ Recipes │ Shelf │ Meals │ Subscriptions  │
└───────────────────────┬─────────────────────────────────────────┘
                        │ HTTPS (JWT via Cognito)
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│              Amazon API Gateway (REST API)                       │
│              Cognito Authorizer (phone + OTP)                    │
└──────┬──────┬──────┬──────┬──────┬──────┬──────┬───────────────┘
       │      │      │      │      │      │      │
       ▼      ▼      ▼      ▼      ▼      ▼      ▼
  auth   user  recipe meal  shelf  voice  bedrock  subscription
  λ      λ     λ      plan  item   λ      ai-λ     λ
                      λ     λ
       │      │      │      │      │      │      │
       └──────┴──────┴──────┴──────┴──────┴──────┘
                        │
              ┌─────────┴──────────┐
              ▼                    ▼
        DynamoDB (12 tables)   Amazon S3
        (users, recipes,       (food photos,
         meals, shelf,          audio cache)
         subscriptions...)
                        │
              ┌─────────┴──────────┐
              ▼                    ▼
        Amazon Bedrock         AWS Transcribe
        Nova Chat/Text/Vision  + AWS Polly
        (AI responses,         (Voice I/O,
         recipe gen,            6 languages)
         food recognition)
```

### Lambda Functions (13)
| Function | Purpose | Memory | Timeout |
|----------|---------|--------|---------|
| auth-handler | Phone OTP auth via Fast2SMS | 256MB | 10s |
| user-handler | User profile CRUD | 256MB | 10s |
| conversation-handler | Chat session management | 256MB | 10s |
| message-handler | Chat message processing | 256MB | 10s |
| recipe-handler | Recipe CRUD + AI generation | 256MB | 10s |
| meal-plan-handler | Meal planning + nutrition | 256MB | 10s |
| nutrition-log-handler | Food logging + calorie tracking | 256MB | 10s |
| shelf-item-handler | Pantry/shelf management | 256MB | 10s |
| voice-handler | STT + TTS + voice AI pipeline | 512MB | 30s |
| bedrock-ai-handler | Bedrock AI orchestration | 512MB | 30s |
| subscription-handler | Razorpay subscriptions + webhooks | 256MB | 15s |
| alarm-handler | Wellness reminders + scheduling | 128MB | 10s |
| admin-handler | Admin dashboard APIs | 256MB | 10s |

### DynamoDB Tables (12)
| Table | Purpose |
|-------|---------|
| kiro-users | User profiles, subscription tier, health goals |
| kiro-auth-tokens | OTP tokens, session management |
| kiro-conversations | AI chat sessions |
| kiro-messages | Chat message history |
| kiro-recipes | Recipe library (user + AI-generated) |
| kiro-meal-plans | Daily/weekly meal plans |
| kiro-nutrition-logs | Food logs with calorie data |
| kiro-shelf-items | Pantry inventory |
| kiro-subscriptions | Razorpay subscription records |
| kiro-alarms | Wellness reminder schedules |
| kiro-consultants | Consultant profiles (legacy) |
| kiro-bookings | Booking records (legacy) |

---

## Authentication Flow

Phone-based OTP authentication — no passwords, no email required.

```
User enters phone number
        ↓
Lambda triggers Fast2SMS OTP delivery
        ↓
Cognito Custom Auth Challenge (define → create → verify)
        ↓
OTP verified → Cognito issues JWT tokens
        ↓
Flutter stores tokens (user-scoped local storage)
        ↓
All API calls use JWT in Authorization header
```

Custom Cognito triggers:
- `cognito-define-auth-challenge` — defines the OTP challenge flow
- `cognito-create-auth-challenge` — generates OTP + calls Fast2SMS
- `cognito-verify-auth-challenge` — validates OTP input

---

## Voice AI Pipeline

```
User speaks (Flutter audio recorder)
        ↓
Audio uploaded to S3 (m4a/wav/webm)
        ↓
voice-handler Lambda triggered
        ↓
AWS Transcribe → text (language auto-detected)
        ↓
bedrock-ai-handler → Amazon Bedrock Nova Chat
        ↓
AI response text
        ↓
AWS Polly → audio (Standard or Neural voice)
        ↓
Audio cached in S3 (24h TTL)
        ↓
Flutter plays response audio
```

Cost optimisations in the voice pipeline:
- Audio response caching (70% cost reduction)
- Response length limiting to 200 words
- Batch transcription (50% cheaper than streaming)
- Tier-based voice quality (Standard for Free, Neural for paid)

---

## Subscription Architecture

```
User selects tier → POST /subscriptions/create
        ↓
subscription-handler creates Razorpay subscription
        ↓
Flutter opens Razorpay checkout (recurring billing)
        ↓
User authorises ₹99/month or ₹499/month
        ↓
POST /subscriptions/verify → signature verified
        ↓
DynamoDB: user tier updated, usage counters reset
        ↓
Monthly: Razorpay webhook → subscription.charged
        ↓
Lambda resets usage counters, extends subscription
```

---

## Cost Engineering

| Optimisation | Saving |
|-------------|--------|
| Graviton2 (arm64) Lambda | 20% compute cost reduction |
| On-demand DynamoDB | Pay-per-request, no idle cost |
| S3 Glacier lifecycle (90 days) | 90% storage cost reduction for old photos |
| API Gateway caching | 60% reduction in Lambda invocations |
| CloudWatch log retention (7d dev / 30d prod) | Reduced log storage cost |
| Voice audio caching (24h) | 70% TTS cost reduction |
| Batch transcription | 50% cheaper than streaming STT |
| Usage limits per tier | Prevents runaway AI costs |

Estimated cost per user per month:
- Free tier: ~$0.10
- Wellness (₹99): ~$0.50 (margin: ₹91)
- FamilyCare (₹499): ~$3.25 (margin: ₹472)

---

## CI/CD Pipeline

```
GitHub Push → GitHub Actions workflow
        ↓
SAM Build (Node.js Lambda packages)
        ↓
SAM Deploy → CloudFormation stack update
        ↓
Lambda functions updated (zero-downtime)
        ↓
Flutter build → APK signed (Android keystore)
        ↓
APK distributed (manual / Play Store)
```

---

## Security Design

- All API endpoints protected by Cognito JWT authorizer
- Phone-based auth — no password storage
- OTP tokens stored in DynamoDB with TTL expiry
- Razorpay credentials in AWS Secrets Manager (never in code)
- S3 bucket: private, pre-signed URLs for food photos
- IAM roles with least-privilege per Lambda function
- User-scoped data isolation (all DynamoDB queries keyed by user_id)
- Cross-user data leakage prevention (user-scoped local storage in Flutter)
- Admin dashboard on separate Cognito user pool (admin-only access)

---

## Screens (Flutter App)

| Screen | Purpose |
|--------|---------|
| login_screen | Phone number entry |
| otp_screen | OTP verification |
| register_screen | New user onboarding |
| home_screen | Dashboard with quick actions |
| chat_screen | AI text chat |
| voice_chat_screen | Voice assistant interface |
| recipes_screen | Recipe library |
| generate_recipe_screen | AI recipe generation |
| recipe_detail_screen | Full recipe view |
| planner_screen | Meal planner |
| add_meal_screen | Add meal to plan |
| food_log_screen | Nutrition log |
| food_capture_screen | Photo-based food logging |
| shelf_screen | Pantry/shelf management |
| health_insights_screen | Health analytics |
| wellness_reminder_screen | Reminder setup |
| subscription_screen | Subscription management |
| subscription_selection_screen | Tier selection + upgrade |
| profile_screen | User profile |
| settings_screen | App settings |

---

## What Makes This Different

**Built for India, not adapted for India.**
- Indian food calorie database (not generic Western data)
- 6 regional languages with natural voice responses
- Indian payment gateway (Razorpay) with UPI + card support
- Phone-based auth (no email required — matches Indian user behaviour)
- Recipes seeded with vegetarian, egg, non-veg, vegan Indian cuisine

**Production-grade serverless architecture.**
- 13 Lambda functions, 12 DynamoDB tables, all on AWS SAM
- Graviton2 arm64 for cost efficiency
- X-Ray tracing, CloudWatch monitoring, structured logging
- Multi-environment support (dev / staging / prod)

**AI that knows your context.**
- Bedrock AI responses are personalised to your shelf, meal history, and goals
- Voice assistant understands intent and can act on it
- Photo recognition identifies Indian food items

**Founder-built, not agency-built.**
- Designed, architected, and shipped solo
- 90+ build versions iterated in production
- Real users, real subscriptions, real data

---

## Roadmap

| Phase | Feature | Status |
|-------|---------|--------|
| Phase 1 | Admin Control Panel | ✅ Complete |
| Phase 2 | Usage Tracking & Limits | ✅ Complete |
| Phase 3 | Voice Action Execution | 🔄 In Progress |
| Phase 4 | Family Health Dashboard (FamilyCare tier) | 📋 Planned |
| Phase 5 | "Hey Cojaisy" Wake Word | 📋 Planned |
| Phase 6 | Play Store Launch | 📋 Planned |

---

## Links

- [Architecture Deep Dive](./architecture.md)
- [AI & Voice Pipeline](./ai-voice.md)
- [Subscription & Monetisation](./subscriptions.md)
- [Security Design](./security.md)
- [Infrastructure & Cost](./infrastructure.md)
