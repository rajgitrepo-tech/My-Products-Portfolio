# Infrastructure & Cost — Deep Dive

> 100% serverless. 100% infrastructure as code. Designed for 78%+ gross margin at scale.

[![SAM](https://img.shields.io/badge/IaC-AWS_SAM_CloudFormation-FF9900?style=flat&logo=amazonaws&logoColor=white)](https://aws.amazon.com/serverless/sam)
[![Graviton](https://img.shields.io/badge/Compute-Graviton2_arm64-FF9900?style=flat&logo=amazonaws&logoColor=white)](https://aws.amazon.com/ec2/graviton)
[![GitHub Actions](https://img.shields.io/badge/CI%2FCD-GitHub_Actions-2088FF?style=flat&logo=githubactions&logoColor=white)](https://github.com/features/actions)

---

## Infrastructure as Code

Every AWS resource — Lambda functions, DynamoDB tables, API Gateway, Cognito, S3, IAM roles, CloudWatch alarms — is defined in a single AWS SAM template. Zero manual console changes. Zero configuration drift.

```
template.yaml
  ├── Parameters
  │   ├── Environment (dev | staging | prod)
  │   ├── LogRetentionDays
  │   └── ReservedConcurrency
  ├── Globals
  │   └── Lambda: runtime, arch, env vars, tracing
  ├── Resources
  │   ├── ApiGateway (RestApi + CognitoAuthorizer + UsagePlan)
  │   ├── CognitoUserPool + UserPoolClient + Triggers
  │   ├── 13 × LambdaFunction (with individual IAM roles)
  │   ├── 12 × DynamoDBTable (on-demand, TTL, GSIs)
  │   ├── S3Bucket (private, lifecycle rules, CORS)
  │   ├── SecretsManagerSecret (Razorpay credentials)
  │   └── CloudWatchAlarms (error rate, throttle, latency)
  └── Outputs
      ├── ApiGatewayUrl
      ├── CognitoUserPoolId
      └── CognitoUserPoolClientId
```

Deploy any environment in a single command:
```bash
sam build && sam deploy --config-env prod
```

---

## AWS Resources

| Resource | Count | Configuration |
|----------|:-----:|--------------|
| Lambda Functions | 13 | Node.js 20.x · arm64 Graviton2 · per-function IAM |
| DynamoDB Tables | 12 | On-demand capacity · TTL on auth tokens · GSIs for lookups |
| API Gateway | 1 | REST API · Cognito authorizer · response caching |
| Cognito User Pools | 2 | App users (phone OTP) + Admin users (MFA) |
| S3 Buckets | 1 | Food photos + voice cache + Glacier lifecycle |
| CloudFront Distributions | 1 | CDN for static assets |
| Secrets Manager Secrets | 1 | Razorpay credentials |
| CloudWatch Log Groups | 13 | One per Lambda · retention per environment |
| IAM Roles | 13 | One per Lambda · least privilege |
| CloudWatch Alarms | 5+ | Error rate · throttle · latency · payment failures |

---

## Cost Engineering Decisions

Every architectural decision was evaluated against cost impact. This is not accidental — it is deliberate.

### Graviton2 (arm64) on All 13 Lambdas
- 20% cheaper than x86 at identical performance
- Node.js 20.x runs natively on arm64 — zero code changes
- Applied uniformly across all functions

### On-Demand DynamoDB (12 Tables)
- Zero provisioned capacity — zero idle cost
- Pay per read/write request unit
- Auto-scales to any traffic spike without configuration
- Switch to provisioned + auto-scaling at 50,000+ users for further savings

### S3 Glacier Lifecycle
- Food photos: Standard storage → Glacier Instant Retrieval after 90 days
- 90% storage cost reduction for photos older than 3 months
- Users rarely access historical food photos

### API Gateway Response Caching
- Cache TTL: 300 seconds on GET endpoints
- Applied to: /recipes, /shelf-items, /meal-plans
- Reduces Lambda invocations by ~60% for read-heavy traffic

### CloudWatch Log Retention Tiers
- dev: 7 days · staging: 14 days · prod: 30 days
- Prevents unbounded log storage accumulation

### Reserved Concurrency Caps
- bedrock-ai-handler: max 10 concurrent (Bedrock service limit protection)
- voice-handler: max 5 concurrent (Transcribe job limit)
- All others: max 20 concurrent
- Prevents runaway costs from traffic spikes or bugs

### Voice Audio Caching
- Polly audio cached in S3 by response content hash (24h TTL)
- ~70% cache hit rate on common health questions
- 70% reduction in Polly API calls

---

## Cost at Scale

### Per-User Monthly Infrastructure Cost

| User Tier | Lambda | DynamoDB | S3 | Bedrock | Transcribe | Polly | Total |
|-----------|:------:|:--------:|:--:|:-------:|:----------:|:-----:|:-----:|
| Free (light AI usage) | $0.02 | $0.03 | $0.01 | $0.01 | $0.002 | $0.001 | **~$0.07** |
| Wellness (moderate) | $0.08 | $0.05 | $0.02 | $0.05 | $0.12 | $0.16 | **~$0.50** |
| FamilyCare (heavy) | $0.20 | $0.10 | $0.05 | $0.25 | $0.60 | $0.80 | **~$2.00** |

### At 1,000 Active Users

| AWS Service | Monthly Cost |
|-------------|:-----------:|
| Lambda (13 functions, Graviton2) | ~$15 |
| DynamoDB (12 tables, on-demand) | ~$20 |
| API Gateway (with caching) | ~$10 |
| Amazon Bedrock (AI queries) | ~$30 |
| AWS Transcribe (voice STT) | ~$25 |
| AWS Polly (voice TTS, cached) | ~$20 |
| S3 + CloudFront | ~$5 |
| CloudWatch + X-Ray | ~$5 |
| **Total infrastructure** | **~$130/month** |

**Gross margin at 1,000 users: ~78%**

---

## CI/CD Pipeline

Zero-downtime deployments on every push to main.

```
Developer pushes to main branch
        ↓
GitHub Actions workflow triggered
        ↓
aws-actions/configure-aws-credentials
  (IAM role via OIDC — no long-lived keys)
        ↓
sam build
  → Installs Lambda dependencies
  → Packages function code
        ↓
sam deploy --config-env prod --no-confirm-changeset
  → CloudFormation changeset created
  → Lambda function code updated
  → Zero downtime — in-flight requests complete normally
        ↓
Flutter APK build (separate workflow)
  → flutter build apk --release
  → Signed with Android keystore (GitHub secret)
  → APK artifact uploaded
```

### Environment Promotion

```
feature branch → dev (manual deploy)
        ↓
PR merge to staging → staging (auto-deploy)
        ↓
PR merge to main → prod (auto-deploy)
```

---

## Multi-Environment Configuration

All environment differences are SAM parameters — no code branches, no environment-specific files.

| Parameter | dev | staging | prod |
|-----------|:---:|:-------:|:----:|
| LogRetentionDays | 7 | 14 | 30 |
| ReservedConcurrency | 5 | 10 | 50 |
| ApiCacheEnabled | false | true | true |
| XRayTracingMode | PassThrough | Active | Active |

---

## Monitoring & Observability

### CloudWatch Dashboards
- Lambda: invocations, errors, duration, throttles — per function
- DynamoDB: consumed capacity, throttled requests, latency
- API Gateway: request count, 4xx/5xx rates, latency p50/p95/p99
- Bedrock: invocation count, latency, error rate

### Custom CloudWatch Metrics
| Metric | Emitted By | Purpose |
|--------|-----------|---------|
| AIQueryCount | bedrock-ai-handler | Track AI usage per tier |
| VoiceQueryCount | voice-handler | Track voice feature adoption |
| RecipeGenerationCount | recipe-handler | Track recipe AI usage |
| PaymentSuccess | subscription-handler | Revenue tracking |
| PaymentFailure | subscription-handler | Churn risk detection |

### CloudWatch Alarms
| Alarm | Threshold | Action |
|-------|-----------|--------|
| Lambda error rate | > 5% over 5 min | SNS notification |
| API Gateway 5xx | > 10/min | SNS notification |
| DynamoDB throttle | Any | SNS notification |
| Bedrock latency p99 | > 10s | SNS notification |

### AWS X-Ray Tracing
End-to-end distributed tracing across API Gateway → Lambda → DynamoDB → Bedrock.
Identifies latency bottlenecks — particularly useful for the voice pipeline where multiple services chain together.
