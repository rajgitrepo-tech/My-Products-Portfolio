# Architecture — Deep Dive

> Serverless-first. Event-driven. Designed to scale from 100 to 10 million users without changing a line of infrastructure code.

[![SAM](https://img.shields.io/badge/IaC-AWS_SAM-FF9900?style=flat&logo=amazonaws&logoColor=white)](https://aws.amazon.com/serverless/sam)
[![Lambda](https://img.shields.io/badge/Compute-13_Lambda_Functions-FF9900?style=flat&logo=awslambda&logoColor=white)](https://aws.amazon.com/lambda)
[![API Gateway](https://img.shields.io/badge/API-Amazon_API_Gateway-FF9900?style=flat&logo=amazonaws&logoColor=white)](https://aws.amazon.com/api-gateway)

---

## Design Principles

| Principle | Decision |
|-----------|---------|
| Serverless-first | No EC2, no containers — Lambda + managed services only |
| Single-region, multi-AZ | ap-south-1 (Mumbai) — lowest latency for Indian users |
| Least-privilege IAM | Each Lambda has its own role with only the permissions it needs |
| User-scoped data isolation | All data keyed by Cognito `user_id` — cross-user access is architecturally impossible |
| Cost-aware by design | Every architectural decision evaluated against cost impact |
| Infrastructure as code | 100% of AWS resources defined in SAM template — no manual console changes |

---

## System Layers

```
┌──────────────────────────────────────────────────────────────────┐
│  PRESENTATION LAYER                                               │
│  Flutter Android App (Dart)                                       │
│  Provider · Amplify Cognito · Razorpay SDK · audioplayers        │
└────────────────────────────┬─────────────────────────────────────┘
                             │ HTTPS + Cognito JWT (every request)
┌────────────────────────────▼─────────────────────────────────────┐
│  API LAYER                                                        │
│  Amazon API Gateway (REST API)                                    │
│  Cognito JWT Authorizer · Request Validation · CORS · Caching    │
└────────────────────────────┬─────────────────────────────────────┘
                             │ Lambda proxy integration
┌────────────────────────────▼─────────────────────────────────────┐
│  COMPUTE LAYER                                                    │
│  13 AWS Lambda Functions                                          │
│  Node.js 20.x · arm64 Graviton2 · per-function IAM roles        │
│  auth · user · recipe · meal-plan · nutrition · shelf            │
│  voice · bedrock-ai · subscription · alarm · admin               │
│  conversation · message                                           │
└──────┬──────────────────────────────────────────┬────────────────┘
       │                                          │
┌──────▼──────────────────┐    ┌──────────────────▼───────────────┐
│  DATA LAYER             │    │  AI LAYER                         │
│  DynamoDB (12 tables)   │    │  Amazon Bedrock                   │
│  On-demand capacity     │    │  Nova Chat · Nova Text            │
│  GSI for phone lookups  │    │  Nova Vision                      │
│  TTL on auth tokens     │    │  AWS Transcribe (STT)             │
│  S3 (photos + cache)    │    │  AWS Polly (TTS)                  │
└─────────────────────────┘    └───────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────────────────┐
│  AUTH LAYER                                                      │
│  Amazon Cognito User Pool (CUSTOM_AUTH flow)                     │
│  3 trigger Lambdas: define → create → verify                     │
│  Fast2SMS for OTP delivery (Indian SMS gateway)                  │
└─────────────────────────────────────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────────────────┐
│  OBSERVABILITY LAYER                                             │
│  CloudWatch Logs · Metrics · Alarms                              │
│  AWS X-Ray (distributed tracing)                                 │
│  Custom metrics: AI usage · payment events · error rates        │
└─────────────────────────────────────────────────────────────────┘
```

---

## API Design

Single API Gateway REST API — resource-based routing, Cognito JWT on all endpoints.

```
/auth
  POST /auth/send-otp          ← no auth (public)
  POST /auth/verify-otp        ← no auth (public)
  POST /auth/refresh

/users
  GET  /users/profile
  PUT  /users/profile

/recipes
  GET  /recipes
  POST /recipes
  GET  /recipes/{id}
  POST /recipes/generate       ← Bedrock Nova Text

/meal-plans
  GET  /meal-plans
  POST /meal-plans
  PUT  /meal-plans/{id}

/nutrition-logs
  GET  /nutrition-logs
  POST /nutrition-logs         ← triggers Nova Vision if photo

/shelf-items
  GET  /shelf-items
  POST /shelf-items
  DELETE /shelf-items/{id}

/voice
  GET  /voice/upload-url       ← pre-signed S3 URL for audio upload
  POST /voice/query            ← full STT + AI + TTS pipeline

/subscriptions
  POST /subscriptions/create
  POST /subscriptions/verify
  POST /subscriptions/cancel
  POST /subscriptions/webhook  ← Razorpay (no auth, HMAC verified)

/admin
  GET  /admin/users
  GET  /admin/analytics
  GET  /admin/health
```

---

## Cognito Custom Auth Flow

Standard Cognito `USER_SRP_AUTH` requires a password. This product uses `CUSTOM_AUTH` — phone + OTP only.

```
App initiates CUSTOM_AUTH
        ↓
cognito-define-auth-challenge Lambda
  → Checks if user exists
  → Tells Cognito: issue a custom challenge
        ↓
cognito-create-auth-challenge Lambda
  → Generates 6-digit OTP
  → Calls Fast2SMS API (Indian SMS delivery)
  → Stores OTP in kiro-auth-tokens (DynamoDB, 5-min TTL)
        ↓
User receives SMS, enters OTP in app
        ↓
cognito-verify-auth-challenge Lambda
  → Reads OTP from DynamoDB
  → Validates input (max 3 attempts)
  → Marks token as used (single-use enforcement)
  → Returns pass/fail to Cognito
        ↓
Cognito issues JWT tokens:
  id_token     → user identity claims
  access_token → API authorization
  refresh_token → silent re-auth
        ↓
Flutter stores tokens in user-scoped local storage
All API calls: Authorization: Bearer {access_token}
```

**Why this matters:** Zero password storage. Zero email dependency. OTP tokens are ephemeral, single-use, and auto-deleted by DynamoDB TTL. The largest class of credential attacks is eliminated by design.

---

## Lambda Function Inventory

| Function | Responsibility | Memory | Timeout | Key Integrations |
|----------|---------------|:------:|:-------:|-----------------|
| auth-handler | Phone OTP + Cognito custom challenge | 256MB | 10s | Cognito, Fast2SMS, DynamoDB |
| user-handler | User profile CRUD + health data | 256MB | 10s | DynamoDB |
| conversation-handler | AI chat session lifecycle | 256MB | 10s | DynamoDB |
| message-handler | Message processing + history | 256MB | 10s | DynamoDB, bedrock-ai-handler |
| recipe-handler | Recipe CRUD + AI generation | 256MB | 10s | DynamoDB, Bedrock Nova Text |
| meal-plan-handler | Meal planning + nutrition calc | 256MB | 10s | DynamoDB |
| nutrition-log-handler | Food logging + photo recognition | 256MB | 10s | DynamoDB, S3, Bedrock Nova Vision |
| shelf-item-handler | Pantry inventory + expiry tracking | 256MB | 10s | DynamoDB |
| voice-handler | Full STT → AI → TTS pipeline | 512MB | 30s | S3, Transcribe, Polly, bedrock-ai-handler |
| bedrock-ai-handler | Context assembly + Bedrock orchestration | 512MB | 30s | DynamoDB, S3, Bedrock Nova Chat/Vision |
| subscription-handler | Razorpay billing + webhook processing | 256MB | 15s | DynamoDB, Razorpay, Secrets Manager |
| alarm-handler | Wellness reminders + scheduling | 128MB | 10s | DynamoDB |
| admin-handler | Operations APIs *(planned)* | 256MB | 10s | DynamoDB, CloudWatch |

All functions: Node.js 20.x · arm64 Graviton2 · least-privilege IAM role

---

## Multi-Environment Strategy

Three fully isolated environments, all defined in `samconfig.toml`:

| Environment | Stack | Log Retention | Reserved Concurrency | Purpose |
|-------------|-------|:-------------:|:-------------------:|---------|
| dev | app-stack-dev | 7 days | 5 | Feature development |
| staging | app-stack-staging | 14 days | 10 | Pre-production validation |
| prod | app-stack-prod | 30 days | 50 | Live users |

All environment-specific config passed as SAM parameters — zero hardcoded values in Lambda code.

---

## Failure Handling

| Failure Scenario | Handling Strategy |
|-----------------|------------------|
| Bedrock throttling | Exponential backoff + retry (3 attempts, jitter) |
| Transcribe job timeout | Error surfaced to user with retry option |
| OTP expired | DynamoDB TTL auto-deletes token, user re-requests |
| Razorpay webhook duplicate | Idempotency check on `subscription_id` before processing |
| Lambda cold start | Provisioned concurrency on prod for auth + bedrock handlers |
| DynamoDB throttle | On-demand capacity auto-scales — no throttle under normal load |
| Cross-user data access | Architecturally prevented — all queries include `user_id` condition |
| S3 upload failure | Pre-signed URL retry logic in Flutter client |

---

## Scalability Path

The serverless architecture scales automatically — but here's the deliberate path:

| Scale | Architecture Change |
|-------|-------------------|
| 0–10K users | Current: on-demand DynamoDB, no provisioned concurrency |
| 10K–100K users | Add DynamoDB auto-scaling, increase reserved concurrency |
| 100K–1M users | Add DynamoDB DAX (caching layer), CloudFront for API responses |
| 1M+ users | Multi-region active-active, DynamoDB Global Tables, Route 53 latency routing |

No re-architecture needed at any scale — just configuration changes.
