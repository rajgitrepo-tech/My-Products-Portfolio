# Security Design

> Zero passwords. Zero email. Least-privilege everywhere. User isolation enforced at the architecture level — not the application level.

[![Cognito](https://img.shields.io/badge/Auth-Amazon_Cognito_OTP-FF9900?style=flat&logo=amazonaws&logoColor=white)](https://aws.amazon.com/cognito)
[![IAM](https://img.shields.io/badge/IAM-Least_Privilege_per_Function-FF9900?style=flat&logo=amazonaws&logoColor=white)](https://aws.amazon.com/iam)
[![Secrets](https://img.shields.io/badge/Secrets-AWS_Secrets_Manager-FF9900?style=flat&logo=amazonaws&logoColor=white)](https://aws.amazon.com/secrets-manager)

---

## Security Philosophy

Security is not a checklist — it is a set of architectural decisions that make the wrong thing structurally impossible.

- No passwords — eliminates credential stuffing, brute force, and password reuse attacks entirely
- No email — eliminates phishing vectors and account enumeration
- No shared IAM roles — each Lambda function can only access what it specifically needs
- No secrets in code or environment variables — all credentials managed by AWS Secrets Manager
- No cross-user data access — user isolation enforced at the data layer, not the application layer

---

## Authentication Model

Phone number + OTP only. No passwords. No email required.

This matches Indian consumer behaviour — the majority of Indian users authenticate via phone across banking, payments, and social apps. It also eliminates the most common attack vectors.

| Property | Design |
|----------|--------|
| Auth method | Phone number + one-time passcode |
| OTP delivery | Indian SMS gateway (high carrier deliverability) |
| OTP lifetime | 5 minutes — auto-expired, never reusable |
| Attempt limit | Maximum 3 attempts before lockout |
| Token standard | Cognito JWT — id token, access token, refresh token |
| Token validation | Validated on every API request before any Lambda executes |
| Token storage | User-scoped local storage on device |

The OTP flow uses Cognito's custom authentication challenge — three Lambda triggers handle the challenge lifecycle. Tokens are ephemeral and single-use by design.

---

## API Security

| Control | Approach |
|---------|---------|
| Transport | HTTPS only — no HTTP |
| Authorization | JWT validated on every request via API Gateway authorizer |
| Webhook integrity | HMAC signature verification on payment callbacks |
| CORS | Restricted to application origins |
| Input validation | Validated at API Gateway and Lambda layers |
| Rate limiting | Per-user throttling via API Gateway usage plans |
| Concurrency limits | Per-function caps prevent resource exhaustion |

---

## User Data Isolation

Enforced at the storage layer — not just application logic. Even if application code had a bug, the data layer prevents cross-user access.

| Layer | Isolation Mechanism |
|-------|-------------------|
| Database | Every query is scoped to the authenticated user's identity — a query without the user identifier cannot return another user's data |
| File storage | All objects stored under user-specific paths — no shared or guessable paths |
| Device storage | Local cache keys are prefixed with user identity — switching accounts clears previous user's data |
| Shared cache | Generic AI responses cached by content hash only — user-specific responses bypass the cache entirely |
| Admin access | Read-only, separate IAM role — cannot modify user data |

---

## IAM — Least Privilege

Each Lambda function has its own dedicated IAM execution role. No shared roles. No wildcard resource permissions.

Functions are granted access only to the specific tables, storage paths, and AWS services they directly use. A function that handles recipes has no access to authentication data. A function that handles voice has no access to payment records.

A compromised function cannot pivot to access unrelated data.

---

## Secrets Management

No credentials exist in source code, configuration files, or environment variables as plaintext.

All sensitive credentials — payment gateway keys, SMS gateway keys, webhook secrets — are stored in AWS Secrets Manager and fetched at runtime. They are never logged, never exported, and never appear in infrastructure templates.

---

## Storage Security

| Asset | Access Model |
|-------|-------------|
| Food photos | Private bucket — accessed via short-lived pre-signed URLs only |
| Voice audio | Private bucket — accessed via short-lived pre-signed URLs only |
| Cached AI responses | Private bucket — no direct public access |
| Static assets | Served via CloudFront — no direct storage access |

All data at rest is encrypted. Pre-signed URLs expire within minutes — there are no permanent public links to user data.

---

## Observability & Incident Response

| Control | Implementation |
|---------|---------------|
| Logging | Structured logs across all Lambda functions — no sensitive data logged |
| Distributed tracing | End-to-end request tracing across API and compute layers |
| Error alerting | Automated alerts on error rate thresholds |
| Anomaly detection | Custom metrics track unusual usage patterns |
| Data isolation testing | Multi-user test scenarios validate isolation on every deployment |

---

## Deliberate Design Decisions

**Phone-only authentication:**
Chosen for Indian market fit — phone-based auth is the dominant pattern across Indian fintech, payments, and consumer apps. The OTP itself serves as a second factor (something you have = your phone). Fallback SMS provider is on the roadmap for resilience.

**Shared response cache:**
Generic health advice responses are cached by content hash. This is safe because these responses contain no personal data. Any response that references a user's personal context — their pantry, their meals, their goals — bypasses the cache entirely and is never stored as shared content.

**No source code published:**
This is a proprietary product. Architecture and design decisions are documented here for professional visibility — implementation details remain private.