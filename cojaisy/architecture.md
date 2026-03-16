# Cojaisy — Architecture Deep Dive

## Design Principles

1. **Serverless-first** — no servers to manage, scale automatically, pay per use
2. **Single-region, multi-AZ** — ap-south-1 (Mumbai) for low latency to Indian users
3. **Least-privilege IAM** — each Lambda has its own role with minimal permissions
4. **User-scoped isolation** — all data keyed by Cognito user_id, no cross-user leakage
5. **Cost-aware design** — every architectural decision evaluated against cost impact

---

## System Layers

```
┌──────────────────────────────────────────────────────────────────┐
│  PRESENTATION LAYER                                               │
│  Flutter Android App (Dart)                                       │
│  Provider state management | Amplify Cognito | Razorpay SDK      │
└────────────────────────────┬─────────────────────────────────────┘
                             │ HTTPS + JWT
┌────────────────────────────▼─────────────────────────────────────┐
│  API LAYER                                                        │
│  Amazon API Gateway (REST)                                        │
│  Cognito JWT Authorizer | Request validation | CORS              │
└────────────────────────────┬─────────────────────────────────────┘
                             │ Lambda proxy integration
┌────────────────────────────▼─────────────────────────────────────┐
│  COMPUTE LAYER                                                    │
│  13 AWS Lambda Functions (Node.js 20.x, arm64/Graviton2)         │
│  auth | user | recipe | meal-plan | nutrition | shelf            │
│  voice | bedrock-ai | subscription | alarm | admin               │
│  conversation | message                                           │
└──────┬──────────────────────────────────────────┬────────────────┘
       │                                          │
┌──────▼──────────────────┐    ┌──────────────────▼───────────────┐
│  DATA LAYER             │    │  AI LAYER                         │
│  DynamoDB (12 tables)   │    │  Amazon Bedrock                   │
│  On-demand capacity     │    │  Nova Chat (conversation)         │
│  GSI for phone lookups  │    │  Nova Text (recipe generation)    │
│  TTL on auth tokens     │    │  Nova Vision (food recognition)   │
│                         │    │  AWS Transcribe (STT)             │
│  S3 (food photos,       │    │  AWS Polly (TTS)                  │
│  audio cache)           │    │                                   │
└─────────────────────────┘    └───────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────────────────┐
│  AUTH LAYER                                                      │
│  Amazon Cognito User Pool                                        │
│  Custom auth challenge (phone + OTP)                             │
│  3 trigger Lambdas: define → create → verify                     │
│  Fast2SMS for OTP delivery (Indian SMS gateway)                  │
└─────────────────────────────────────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────────────────┐
│  OBSERVABILITY LAYER                                             │
│  CloudWatch Logs (13 log groups, 7d dev / 30d prod retention)   │
│  CloudWatch Metrics (custom: PaymentSuccess, SubscriptionChurn) │
│  AWS X-Ray (distributed tracing across Lambda + API Gateway)    │
│  AWS Secrets Manager (Razorpay credentials)                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## API Design

Single API Gateway REST API with resource-based routing:

```
/auth
  POST /auth/send-otp
  POST /auth/verify-otp
  POST /auth/refresh

/users
  GET  /users/profile
  PUT  /users/profile

/recipes
  GET  /recipes
  POST /recipes
  GET  /recipes/{id}
  POST /recipes/generate        ← Bedrock AI generation

/meal-plans
  GET  /meal-plans
  POST /meal-plans
  PUT  /meal-plans/{id}

/nutrition-logs
  GET  /nutrition-logs
  POST /nutrition-logs

/shelf-items
  GET  /shelf-items
  POST /shelf-items
  DELETE /shelf-items/{id}

/voice
  POST /voice/query             ← STT + AI + TTS pipeline

/subscriptions
  POST /subscriptions/create
  POST /subscriptions/verify
  POST /subscriptions/cancel
  POST /subscriptions/webhook   ← Razorpay webhook (no auth)

/admin
  GET  /admin/users
  GET  /admin/analytics
  GET  /admin/health
```

---

## Data Model (Key Tables)

### kiro-users
```
user_id (PK)          ← Cognito sub
phone_number          ← GSI PK for phone lookups
name
age, gender
dietary_preference    ← vegetarian | egg | non-veg | vegan
health_goals          ← weight_loss | muscle_gain | maintenance
subscription_tier     ← free | wellness | familycare
razorpay_subscription_id
subscription_start_date
subscription_end_date
voice_count_month     ← usage tracking
photo_count_month
recipe_count_month
usage_reset_date
created_at
```

### kiro-recipes
```
recipe_id (PK)
user_id (GSI)
name
ingredients           ← JSON array
instructions          ← JSON array
nutrition             ← calories, protein, carbs, fat
dietary_type          ← vegetarian | egg | non-veg | vegan
cuisine               ← Indian regional
is_ai_generated       ← boolean
created_at
```

### kiro-auth-tokens
```
token_id (PK)
phone_number
otp_code
expires_at            ← TTL attribute (DynamoDB auto-delete)
attempts
verified
```

---

## Multi-Environment Strategy

SAM config (`samconfig.toml`) defines three environments:

| Environment | Stack Name | Log Retention | Reserved Concurrency |
|-------------|-----------|---------------|---------------------|
| dev | kiro-stack-dev | 7 days | 5 |
| staging | kiro-stack-staging | 14 days | 10 |
| prod | kiro-stack-prod | 30 days | 50 |

All environment-specific values passed as SAM parameters — no hardcoded values in Lambda code.

---

## Cognito Custom Auth Flow

Standard Cognito USER_SRP_AUTH doesn't support phone-only OTP.
Cojaisy uses `CUSTOM_AUTH` flow with three trigger Lambdas:

```
1. cognito-define-auth-challenge
   → Tells Cognito: "use custom challenge"
   → Checks if user exists, initiates challenge

2. cognito-create-auth-challenge
   → Generates 6-digit OTP
   → Calls Fast2SMS API to deliver SMS
   → Stores OTP in DynamoDB (kiro-auth-tokens) with 5-min TTL

3. cognito-verify-auth-challenge
   → Reads OTP from DynamoDB
   → Compares with user input
   → Marks token as verified
   → Returns success/fail to Cognito
```

On success, Cognito issues:
- `id_token` (JWT with user claims)
- `access_token` (for API calls)
- `refresh_token` (for silent re-auth)

---

## Bedrock AI Integration

The `bedrock-ai-handler` Lambda orchestrates all AI calls:

```javascript
// Nova Chat — conversational AI
const response = await bedrock.invokeModel({
  modelId: 'amazon.nova-chat-v1',
  body: JSON.stringify({
    messages: conversationHistory,
    system: systemPrompt,  // includes user's shelf, goals, dietary prefs
    inferenceConfig: { maxTokens: 500, temperature: 0.7 }
  })
});

// Nova Vision — food photo recognition
const response = await bedrock.invokeModel({
  modelId: 'amazon.nova-vision-v1',
  body: JSON.stringify({
    messages: [{
      role: 'user',
      content: [
        { image: { format: 'jpeg', source: { bytes: imageBase64 } } },
        { text: 'Identify this Indian food item and estimate calories' }
      ]
    }]
  })
});
```

System prompt is dynamically constructed per user:
- Dietary preference (vegetarian/non-veg/vegan/egg)
- Health goals (weight loss / muscle gain / maintenance)
- Current shelf items (for recipe suggestions)
- Recent meal history (for nutritional context)

---

## Voice Pipeline Architecture

```
Flutter AudioRecorder
  → records m4a audio
  → uploads to S3: voice-input/{user_id}/{timestamp}.m4a

voice-handler Lambda
  → reads S3 object
  → detects audio format (m4a/wav/webm/ogg)
  → calls AWS Transcribe (batch job)
  → polls for completion
  → extracts transcript text

  → calls bedrock-ai-handler (internal invoke)
  → gets AI response text

  → checks S3 audio cache (hash of response text)
  → if cache miss: calls AWS Polly
    → Standard voice (Free tier)
    → Neural voice (Wellness/FamilyCare)
  → stores audio in S3 cache (24h TTL)

  → returns: transcript + response text + audio S3 URL

Flutter
  → displays transcript
  → plays audio via audioplayers package
```

Language detection is automatic via Transcribe's `IdentifyLanguage` feature,
with a hint list of the 6 supported Indian language codes.

---

## Admin Dashboard Architecture

```
CloudFront Distribution
  → S3 bucket (React build, static assets)
  → Custom domain (optional)

React App
  → Cognito (admin-only user pool, MFA enabled)
  → API Gateway /admin/* endpoints
  → admin-handler Lambda

admin-handler Lambda
  → DynamoDB: scan users, aggregate stats
  → CloudWatch: pull Lambda metrics, error rates
  → API Gateway: check endpoint health
  → Bedrock: check model availability
  → Returns: user counts, revenue, AI usage, system health
```

Admin dashboard components:
- Dashboard (KPI overview)
- UserList / UserDetail (user management)
- SubscriptionList / SubscriptionDetail / SubscriptionAnalytics
- UserAnalytics / AIUsageAnalytics / InfrastructureCostAnalytics
- LambdaHealth / DynamoDBHealth / BedrockHealth / APIGatewayHealth
- AuditLogs

---

## Failure Handling

| Scenario | Handling |
|----------|---------|
| Bedrock throttling | Exponential backoff + retry (3 attempts) |
| Transcribe job failure | Error returned to user with retry option |
| Razorpay webhook duplicate | Idempotency check on subscription_id |
| OTP expired | DynamoDB TTL auto-deletes, user must re-request |
| Lambda cold start | Provisioned concurrency on prod for auth + bedrock handlers |
| DynamoDB throttle | On-demand capacity auto-scales, no throttle under normal load |
| Cross-user data leakage | All queries include user_id condition, Flutter uses user-scoped storage |
