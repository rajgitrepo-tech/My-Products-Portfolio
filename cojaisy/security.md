# Cojaisy — Security Design

## Authentication & Identity

| Control | Implementation |
|---------|---------------|
| Auth method | Phone number + OTP (no passwords) |
| OTP delivery | Fast2SMS (Indian SMS gateway) |
| OTP storage | DynamoDB with 5-minute TTL (auto-deleted) |
| OTP attempts | Max 3 attempts before lockout |
| Token type | Cognito JWT (id_token + access_token + refresh_token) |
| Token validation | API Gateway Cognito authorizer (every request) |
| Token refresh | Silent refresh via Amplify SDK |
| Admin auth | Separate Cognito user pool (admin-only, MFA enabled) |

## API Security

| Control | Implementation |
|---------|---------------|
| Transport | HTTPS only (TLS 1.2+) |
| Authorization | Cognito JWT on all endpoints except /auth and /subscriptions/webhook |
| Webhook auth | HMAC-SHA256 signature verification (Razorpay secret) |
| CORS | Restricted to app origins |
| Input validation | API Gateway request validators + Lambda-level validation |
| Rate limiting | API Gateway usage plans (per-user throttling) |

## Data Isolation

| Control | Implementation |
|---------|---------------|
| DynamoDB queries | All queries include `user_id` condition — no cross-user data possible |
| S3 objects | Keyed by `{user_id}/{timestamp}` — no shared paths |
| Flutter storage | User-scoped SharedPreferences (keys prefixed with user_id) |
| Cache isolation | Audio cache keyed by content hash, not user — safe for shared responses |
| Admin access | Admin Lambda uses separate IAM role, cannot access user data directly |

## Secrets Management

| Secret | Storage |
|--------|---------|
| Razorpay key_id + key_secret | AWS Secrets Manager |
| Razorpay webhook secret | AWS Secrets Manager |
| Fast2SMS API key | AWS Secrets Manager |
| Cognito User Pool ID / Client ID | SAM parameters (CloudFormation outputs) |
| No secrets in code | Enforced — all Lambda env vars reference Secrets Manager ARNs |

## IAM — Least Privilege

Each Lambda function has its own IAM execution role with only the permissions it needs:

| Lambda | DynamoDB | S3 | Bedrock | Transcribe | Polly | Secrets |
|--------|----------|-----|---------|-----------|-------|---------|
| auth-handler | kiro-auth-tokens (rw), kiro-users (r) | ✗ | ✗ | ✗ | ✗ | Fast2SMS key |
| user-handler | kiro-users (rw) | ✗ | ✗ | ✗ | ✗ | ✗ |
| recipe-handler | kiro-recipes (rw), kiro-shelf-items (r) | ✗ | Nova Text | ✗ | ✗ | ✗ |
| bedrock-ai-handler | kiro-users (r), kiro-messages (rw) | food-photos (r) | Nova Chat, Nova Vision | ✗ | ✗ | ✗ |
| voice-handler | kiro-users (r) | voice-input (rw), voice-cache (rw) | ✗ | StartJob, GetJob | SynthSpeech | ✗ |
| subscription-handler | kiro-subscriptions (rw), kiro-users (rw) | ✗ | ✗ | ✗ | ✗ | Razorpay keys |
| admin-handler | All tables (r) | ✗ | ✗ | ✗ | ✗ | ✗ |

## S3 Security

- Bucket: private (no public access)
- Food photos: accessed via pre-signed URLs (5-minute expiry)
- Voice audio cache: accessed via pre-signed URLs (5-minute expiry)
- Admin dashboard: public read (static website hosting) — no sensitive data
- Lifecycle rules: food photos moved to Glacier after 90 days

## Monitoring & Incident Response

| Control | Implementation |
|---------|---------------|
| Logging | CloudWatch Logs for all 13 Lambda functions |
| Tracing | AWS X-Ray (Lambda + API Gateway) |
| Metrics | Custom CloudWatch metrics: PaymentSuccess, PaymentFailure, SubscriptionChurn |
| Alerts | CloudWatch Alarms on Lambda error rate > 5% |
| Audit | Admin dashboard AuditLogs component tracks admin actions |
| Data breach detection | Cross-user data leakage tests in CI (multi-user test suite) |

## Known Security Decisions

**Phone-only auth (no email):**
Matches Indian user behaviour — most users don't have or use email for apps.
OTP via SMS is the standard auth pattern for Indian consumer apps.
Trade-off: SMS delivery depends on Fast2SMS uptime.

**No password storage:**
Eliminates the largest class of credential-based attacks.
OTP tokens are ephemeral (5-min TTL) and single-use.

**Shared audio cache:**
Voice responses are cached by content hash, not user_id.
This means two users asking the same question share the same cached audio.
This is intentional — audio responses contain no PII (they're generic health advice).
User-specific responses (e.g., "your shelf has X items") are not cached.
