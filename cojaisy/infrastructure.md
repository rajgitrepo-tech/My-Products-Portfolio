# Cojaisy — Infrastructure & Cost

## Infrastructure as Code

All AWS resources defined in a single AWS SAM template (`template.yaml`).
SAM extends CloudFormation with Lambda-specific shorthand.

```
template.yaml
  ├── Parameters (env, region, log retention)
  ├── Globals (Lambda runtime, arch, env vars)
  ├── Resources
  │   ├── API Gateway (RestApi + Authorizer)
  │   ├── Cognito User Pool + Client + Triggers
  │   ├── 13 Lambda Functions
  │   ├── 12 DynamoDB Tables
  │   ├── S3 Bucket (food photos + audio cache)
  │   ├── Secrets Manager Secret (Razorpay)
  │   └── IAM Roles (one per Lambda)
  └── Outputs (API URL, Cognito IDs, S3 bucket name)
```

Deploy command:
```bash
sam build && sam deploy --config-env prod
```

---

## AWS Resources Summary

| Resource Type | Count | Notes |
|--------------|-------|-------|
| Lambda Functions | 13 | Node.js 20.x, arm64 (Graviton2) |
| DynamoDB Tables | 12 | On-demand capacity |
| API Gateway | 1 | REST API, Cognito authorizer |
| Cognito User Pools | 2 | App users + Admin users |
| S3 Buckets | 1 | Food photos + voice cache + admin dashboard |
| CloudFront Distributions | 1 | Admin dashboard CDN |
| Secrets Manager Secrets | 1 | Razorpay credentials |
| CloudWatch Log Groups | 13 | One per Lambda |
| IAM Roles | 13 | One per Lambda (least privilege) |

---

## Cost Optimisation Decisions

### Graviton2 (arm64) Lambda
- 20% cheaper than x86 Lambda
- Same Node.js 20.x runtime, no code changes needed
- Applied to all 13 Lambda functions

### On-Demand DynamoDB
- No provisioned capacity to manage
- Pay per read/write unit
- Cost-effective at current scale (< 10,000 users)
- Switch to provisioned + auto-scaling at 50,000+ users

### S3 Glacier Lifecycle
- Food photos: Standard → Glacier after 90 days
- 90% storage cost reduction for old photos
- Users rarely access photos older than 30 days

### API Gateway Caching
- Cache TTL: 300 seconds for GET endpoints
- Reduces Lambda invocations by ~60% for read-heavy endpoints
- Applied to: /recipes, /shelf-items, /meal-plans

### CloudWatch Log Retention
- Dev: 7 days
- Staging: 14 days
- Prod: 30 days
- Prevents unbounded log storage costs

### Reserved Concurrency
- Prevents Lambda from scaling beyond budget
- bedrock-ai-handler: max 10 concurrent (Bedrock rate limit protection)
- voice-handler: max 5 concurrent (Transcribe job limit)
- Other functions: max 20 concurrent

---

## Cost Estimates (Production)

### Per-User Monthly Cost

| User Type | Lambda | DynamoDB | S3 | Bedrock | Transcribe | Polly | Total |
|-----------|--------|----------|-----|---------|-----------|-------|-------|
| Free (2 AI queries) | $0.02 | $0.03 | $0.01 | $0.01 | $0.002 | $0.001 | **~$0.07** |
| Wellness (100 voice) | $0.08 | $0.05 | $0.02 | $0.05 | $0.12 | $0.16 | **~$0.50** |
| FamilyCare (500 voice) | $0.20 | $0.10 | $0.05 | $0.25 | $0.60 | $0.80 | **~$2.00** |

### At 1,000 Users (Mixed Tier)
Assuming 70% Free, 25% Wellness, 5% FamilyCare:

| Component | Monthly Cost |
|-----------|-------------|
| Lambda (13 functions) | ~$15 |
| DynamoDB (12 tables) | ~$20 |
| API Gateway | ~$10 |
| Bedrock (AI queries) | ~$30 |
| Transcribe (voice STT) | ~$25 |
| Polly (voice TTS) | ~$20 |
| S3 + CloudFront | ~$5 |
| CloudWatch | ~$5 |
| **Total** | **~$130/month** |

Monthly Revenue at 1,000 users (same mix):
- 250 Wellness × ₹99 = ₹24,750 (~$300)
- 50 FamilyCare × ₹499 = ₹24,950 (~$300)
- **Total MRR: ~$600**
- **Infrastructure cost: ~$130**
- **Gross margin: ~78%**

---

## CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml (simplified)

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1
      - run: sam build
      - run: sam deploy --config-env prod --no-confirm-changeset
```

Lambda updates are zero-downtime — CloudFormation updates function code
without interrupting in-flight requests.

---

## Multi-Environment Setup

| Environment | Purpose | Deploy Trigger |
|-------------|---------|---------------|
| dev | Local development + feature testing | Manual |
| staging | Pre-prod validation | PR merge to staging branch |
| prod | Live users | Merge to main |

SAM parameters per environment:
- `LogRetentionDays`: 7 / 14 / 30
- `ReservedConcurrency`: 5 / 10 / 50
- `Environment`: dev / staging / prod

All environment-specific config passed as CloudFormation parameters —
no environment-specific code branches.

---

## Monitoring Setup

### CloudWatch Dashboards
- Lambda invocations, errors, duration per function
- DynamoDB read/write capacity consumed
- API Gateway 4xx/5xx error rates
- Bedrock invocation count and latency

### Custom Metrics
- `PaymentSuccess` — Razorpay subscription charged
- `PaymentFailure` — Razorpay payment failed
- `SubscriptionCancellation` — User cancelled
- `VoiceQueryCount` — Voice assistant usage
- `RecipeGenerationCount` — AI recipe generation usage

### Alarms
- Lambda error rate > 5% → SNS notification
- API Gateway 5xx > 10/minute → SNS notification
- DynamoDB throttle events → SNS notification

### X-Ray Tracing
- End-to-end request tracing: API Gateway → Lambda → DynamoDB / Bedrock
- Identifies latency bottlenecks in the voice pipeline
- Bedrock invocation latency tracked separately
