# Enterprise GenAI Platform on AWS — Final Plan

## Goal

Build a **multi-tenant enterprise LLM platform** that allows internal teams to access Amazon Bedrock through a unified gateway with:

* Tenant isolation
* Authentication and authorization
* Quotas and rate limits
* Guardrails
* Model routing and fallback
* Capacity management
* Caching
* Streaming
* Async workloads
* Observability
* FinOps
* Evaluation
* Model lifecycle management
* Governance and onboarding

The goal is not to build a simple Bedrock wrapper, but a production-grade internal AI platform.

---

# 1. Core Architecture Principles

The platform should enforce four core invariants.

```text
1. Isolation Invariant

Tenant A cannot access or interfere with:
- Tenant B data
- Tenant B policy
- Tenant B quota
- Tenant B cache
- Tenant B logs


2. Safety Invariant

If a required safety or guardrail check is unavailable,
the request must not reach the model.


3. Policy Invariant

Tenant disablement, policy revocation, and security policy
updates must become effective within a bounded amount of time.


4. Routing Invariant

Production traffic may only be routed to models that have
passed the required evaluation and certification gates.
```

---

# 2. High-Level Architecture

```text
                 ┌─────────────────────────────┐
                 │        CONTROL PLANE        │
                 │                             │
                 │ Tenant Onboarding           │
                 │ Tenant State / Kill Switch  │
                 │ Model Allowlist             │
                 │ Quota / Budget              │
                 │ Guardrail Policy            │
                 │ Prompt / Model Version      │
                 │ Evaluation / Certification  │
                 │ Cost / Usage                │
                 │ Approval / Governance       │
                 └─────────────┬───────────────┘
                               │
                    versioned policy snapshot
                               │
                               ▼
                     Regional Policy Cache
                               │
───────────────────────────────┼──────────────────────────────
                               │
                        DATA PLANE
                               │
Client                         │
  │                            │
  │ OIDC / OAuth2              │
  ▼
WAF → ALB
      │
      ▼
┌────────────────────────────────────────────────────┐
│                  LLM Gateway                       │
│                                                    │
│ AuthN                                              │
│   ↓                                                │
│ AuthZ / RBAC / ABAC                                │
│   ↓                                                │
│ Tenant Resolution                                  │
│   ↓                                                │
│ Emergency Kill Switch                              │
│   ↓                                                │
│ Policy Snapshot / Epoch                            │
│   ↓                                                │
│ Rate Limit / Quota / Budget                        │
│   ↓                                                │
│ Input Guardrail                                    │
│   ↓                                                │
│ Cache                                              │
│   ↓                                                │
│ Certified Model Router                             │
│   ↓                                                │
│ Retry / Timeout / Circuit Breaker                  │
│   ↓                                                │
│ Bedrock                                            │
│   ↓                                                │
│ Output Guardrail                                   │
│   ↓                                                │
│ Cache Write                                        │
│   ↓                                                │
│ SSE Streaming Response                             │
│                                                    │
│ → Telemetry → Audit → Cost                         │
└───────────────────────┬────────────────────────────┘
                        │
          ┌─────────────┼────────────────────┐
          ▼             ▼                    ▼
     On-Demand     Cross-Region        Provisioned
      Bedrock       Inference          Throughput

                        │
             ┌──────────┴───────────┐
             │                      │
          Async                 Offline Batch
             │                      │
            SQS                Bedrock Batch
             │                      │
        ECS Worker                  S3
```

---

# 3. Compute / Inference

Initial compute platform:

```text
ECS Fargate
```

Core services:

```text
gateway-api
worker
admin-api
```

Avoid splitting every feature into a separate microservice in the first version.

Bedrock is the main inference backend.

Supported capacity paths:

```text
Router
 ├── Regional On-Demand
 ├── Cross-Region Inference Profile
 └── Provisioned Throughput
```

These are separate capacity strategies.

Do not assume:

```text
Provisioned Throughput
        ↓
Cross-Region Inference
```

Capacity management must handle:

```text
Bedrock quotas
429 throttling
traffic bursts
retry
exponential backoff
jitter
circuit breaker
capacity saturation
```

---

# 4. Gateway API

Core APIs:

```text
POST /v1/chat
POST /v1/embeddings
POST /v1/batch

GET /v1/jobs/{job_id}
GET /v1/models
```

Admin APIs:

```text
POST /v1/admin/tenants

PUT /v1/admin/tenants/{id}/quota
PUT /v1/admin/tenants/{id}/models
PUT /v1/admin/tenants/{id}/guardrails
PUT /v1/admin/tenants/{id}/state

GET /v1/admin/usage
```

API governance:

```text
OpenAPI specification
/v1 /v2 versioning
idempotency keys
standard tool schema
function-calling schema
request_id
standardized error model
```

---

# 5. Authentication and Authorization

Authentication flow:

```text
OIDC / OAuth2
      ↓
JWT
      ↓
Authentication
      ↓
Identity
      ↓
tenant_id
      ↓
RBAC / ABAC
```

Do not trust arbitrary client-provided tenant headers such as:

```http
X-Tenant-ID: finance
```

Tenant identity must be derived from authenticated identity.

Example JWT claims:

```json
{
  "sub": "user123",
  "tenant_id": "finance",
  "roles": ["developer"],
  "application_id": "risk-chat"
}
```

Service-to-service authorization uses:

```text
IAM
STS
AssumeRole
```

---

# 6. Multi-Tenancy

Every request should carry:

```text
tenant_id
application_id
request_id
```

Example tenant policy:

```yaml
tenant: finance

state: ACTIVE

models:
  - claude-sonnet
  - nova-pro

rpm_limit: 300
tpm_limit: 500000

monthly_budget: 10000

guardrail_policy: finance-v7

route_set:
  finance-chat-v12

slo:
  p95_latency_ms: 3000

policy_epoch: 42
```

Tenant-specific configuration may include:

```text
model allowlist
quota
rate limits
guardrails
budget
fallback policy
SLO
data retention
logging policy
```

---

# 7. Tenant State and Kill Switch

Tenant state should be a first-class platform primitive.

Possible states:

```text
ACTIVE
THROTTLED
READ_ONLY
SUSPENDED
EMERGENCY_BLOCK
```

Request path:

```text
Auth
 ↓
tenant_id
 ↓
Emergency Gate
      │
      ├── ACTIVE → continue
      │
      └── BLOCK → reject
```

The kill switch must execute before:

```text
cache
budget processing
model routing
fallback
inference
```

---

# 8. Tenant Policy Consistency

Tenant policy may be stored in DynamoDB, but the data plane should not synchronously fetch control-plane configuration for every request.

Preferred architecture:

```text
Control Plane
      ↓
DynamoDB / Global Table
      ↓
Policy Change Event
      ↓
Regional Policy Cache
      ↓
Gateway
```

Each cached policy contains:

```text
tenant_policy
policy_epoch
expires_at
```

Use:

```text
push-based invalidation
+
bounded TTL / lease
```

Example:

```text
Event propagation target: < 1–2 seconds

Policy lease: 30 seconds
```

This means:

```text
Maximum stale-policy lifetime <= 30 seconds
```

Principle:

> Push invalidation provides low propagation latency, while the TTL or lease provides a correctness bound.

---

# 9. Control Plane Availability

The data plane must not synchronously depend on the control plane being available.

Bad design:

```text
Request
 ↓
Control Plane API
 ↓
Can this tenant call the model?
```

If the control plane fails, all inference fails.

Preferred model:

```text
Control Plane
     ↓
Publish
     ↓
Regional Policy Snapshot
     ↓
Data Plane
```

Potential control-plane components:

```text
DynamoDB Global Tables
multi-region admin services
regional event propagation
regional policy cache
```

Consider logically separating:

```text
TenantConfig
```

from:

```text
TenantSafetyState
```

because emergency safety state may require stronger consistency and faster propagation.

---

# 10. Guardrails

Request pipeline:

```text
Input
 ↓
PII / moderation / prompt-injection checks
 ↓
ApplyGuardrail
 ↓
LLM
 ↓
ApplyGuardrail
 ↓
Output
```

Each tenant may define:

```text
guardrail policy
guardrail version
safety class
allowed models
data handling rules
```

Telemetry should record:

```text
guardrail_version
guardrail_latency
guardrail_action
blocked_reason
```

---

# 11. Guardrail Fail-Closed Behavior

Define the following invariant:

```text
No request requiring a strong guardrail may reach
the LLM unless the required guardrail decision
has completed successfully.
```

Failure behavior:

```text
Guardrail
   │
   ├── ALLOW → model
   ├── BLOCK → reject
   │
   ├── timeout
   ├── 429
   ├── 500
   └── unavailable
          ↓
     bounded retry
          ↓
    still unavailable
          ↓
      FAIL CLOSED
```

Example response:

```text
HTTP 503

AI_SAFETY_SERVICE_UNAVAILABLE
```

Do not silently downgrade from:

```text
Strong Guardrail
```

to:

```text
Weak Guardrail
```

unless the tenant policy explicitly allows the downgrade.

Possible safety classes:

```text
STRICT
STANDARD
LOW_RISK
```

---

# 12. Cache Design

Cache lookup should happen only after tenant and safety checks.

Correct ordering:

```text
Auth
 ↓
Tenant State
 ↓
Policy
 ↓
Input Guardrail
 ↓
Cache Lookup
```

Do not perform cache lookup before tenant or policy enforcement.

Recommended cache key:

```text
SHA256(
    tenant_id
  + application_id
  + model_route_id
  + model_version
  + inference_parameters
  + prompt_template_version
  + normalized_messages
  + tool_schema_version
  + guardrail_version
  + policy_epoch
  + retrieval_context_hash
)
```

Important fields:

```text
tenant_id
guardrail_version
policy_epoch
```

Example policy update:

```text
policy_epoch

42 → 43
```

Old cache entries become automatic misses.

This avoids scanning Redis and deleting large numbers of stale keys.

Only cache:

```text
responses that have already passed output guardrails
```

Do not cache raw model output.

Supported cache types:

```text
response cache
semantic cache
prompt-template cache
```

---

# 13. Certified Model Routing

Do not treat fallback as a simple secondary model.

Use a:

```text
Certified Route Set
```

Example:

```yaml
route_set: finance-chat-v12

primary:
  model: claude-sonnet

fallbacks:
  - nova-pro
  - claude-haiku

requirements:
  streaming: true
  tool_calling: true
  structured_output: true

evaluation:
  quality: ">=0.88"
  safety: ">=0.99"
  p95_latency: "<3000ms"
```

Runtime routing:

```text
Primary
   ↓ failure
Certified Fallback #1
   ↓ failure
Certified Fallback #2
```

Models that have not passed evaluation must not receive production traffic, including fallback traffic.

---

# 14. Streaming

Use SSE for the initial `/chat` streaming interface.

```text
Browser
   │
   │ SSE
   ▼
Gateway
   │
   │ ConverseStream
   ▼
Bedrock
```

Gateway must detect:

```text
client disconnect
timeout
cancel request
slow consumer
backpressure
```

On disconnect:

```text
client disconnect
      ↓
abort signal
      ↓
cancel upstream Bedrock request
      ↓
stop stream processing
      ↓
release resources
```

Telemetry:

```text
client_disconnected
upstream_abort
generated_tokens_before_abort
stream_duration
```

---

# 15. Async and Long-Running Workloads

Interactive request:

```text
Gateway
 ↓
Bedrock Converse
```

Async workflow:

```text
POST /jobs
   ↓
SQS
   ↓
ECS Worker
   ↓
Bedrock
   ↓
Result Store
```

Complex orchestration:

```text
Step Functions
```

Large offline workloads:

```text
Input S3
   ↓
Bedrock Batch
   ↓
Output S3
```

---

# 16. Reliability

Gateway reliability mechanisms:

```text
timeouts
bounded retries
exponential backoff
jitter
retry budget
circuit breaker
fallback
bulkhead isolation
rate limiting
DLQ
multi-AZ
```

Avoid retry amplification.

Bad behavior:

```text
Bedrock overloaded
       ↓
429
       ↓
every gateway retries immediately
       ↓
even more overload
```

Use:

```text
bounded retry attempts
exponential backoff
jitter
retry budget
circuit breaker
```

---

# 17. Load Testing and Failure Testing

Load testing is a release gate, not merely a repository folder.

Required scenarios:

```text
baseline capacity
quota saturation
traffic burst
Bedrock 429
Guardrail 429
Guardrail timeout
latency spike
circuit breaker
fallback
stream disconnect
cache failure
Redis unavailable
DynamoDB throttling
tenant noisy neighbor
kill-switch propagation
policy update under load
queue backlog
```

Important path to validate:

```text
429
 ↓
retry + jitter
 ↓
no retry storm
 ↓
circuit breaker
 ↓
certified fallback
```

Testing should include real Bedrock quota behavior where practical.

---

# 18. Observability

Do not limit observability to:

```text
CPU
memory
```

Important AI platform metrics:

```text
RPS
success rate
429 rate
5xx rate

TTFT
P50 latency
P95 latency
P99 latency

input tokens
output tokens
tokens/sec

cost/request
cost/tenant
cost/model

guardrail latency
guardrail block rate

fallback rate
retry count
circuit breaker state

queue depth
queue wait

cache hit rate
stream abort rate
```

Example telemetry record:

```json
{
  "request_id": "req-123",
  "tenant_id": "finance",
  "application_id": "risk-chat",
  "route_set": "finance-chat-v12",
  "model": "claude-sonnet",
  "policy_epoch": 43,
  "guardrail_version": "finance-v7",
  "input_tokens": 1200,
  "output_tokens": 340,
  "ttft_ms": 420,
  "latency_ms": 2200,
  "retry_count": 0,
  "fallback": false,
  "cache_hit": false,
  "status": 200,
  "estimated_cost": 0.014
}
```

Observability stack:

```text
OpenTelemetry
CloudWatch
Prometheus
Grafana
```

---

# 19. PII-Safe Logging and Audit

Default telemetry should store:

```text
metadata only
```

Do not store by default:

```text
raw prompt
raw response
PII
retrieved documents
```

If a tenant explicitly enables debug capture:

```text
Raw Payload
    ↓
PII Detection
    ↓
Redaction
    ↓
Data Classification
    ↓
Encryption
    ↓
Restricted Debug Store
    ↓
Short Retention TTL
```

Separate:

```text
Operational Telemetry
```

from:

```text
Conversation / Debug Storage
```

They should have separate:

```text
IAM permissions
retention policies
encryption policies
access controls
```

---

# 20. FinOps

Track:

```text
tenant
application
model
input tokens
output tokens
request cost
monthly spend
budget utilization
```

Support:

```text
showback
chargeback
budget alerts
cost anomaly detection
```

Example:

```text
Finance      $12,300
Search        $8,200
Fraud         $4,100
```

Budget policy may support:

```text
soft budget limit
hard budget limit
quota throttling
request rejection
```

---

# 21. Evaluation and Model Lifecycle

Models must not be deployed merely by changing:

```text
model_id
```

Lifecycle:

```text
New Model / Prompt
       ↓
Golden Dataset
       ↓
Quality Evaluation
       ↓
Safety Evaluation
       ↓
Tool Calling Evaluation
       ↓
Structured Output Evaluation
       ↓
Latency Test
       ↓
Cost Test
       ↓
Load Test
       ↓
CERTIFIED
       ↓
Canary
5% → 25% → 100%
```

Regression failure:

```text
rollback
```

Example release gate:

```text
Quality >= 0.88
Safety >= 0.99
P95 latency < 3000 ms
Cost/request < $0.02
```

Fallback models must pass the same certification process.

---

# 22. Tenant Onboarding

Final self-service portal:

```text
Enterprise AI Portal

Applications
Models
Quota
Usage
Cost
Guardrails
Prompts
Evaluation
Audit
```

Tenant onboarding workflow:

```text
Team
 ↓
Create Application
 ↓
Request Model
 ↓
Request TPM / RPM
 ↓
Select Data / Safety Policy
 ↓
Security Approval
 ↓
Budget Approval
 ↓
Provision Tenant Policy
 ↓
Ready
```

---

# 23. Infrastructure

Terraform manages:

```text
VPC
public/private subnets
ALB
WAF
ECS
ECR
IAM
STS
KMS
Secrets Manager
DynamoDB
Redis
SQS
S3
CloudWatch
Bedrock VPC endpoints
```

Environment structure:

```text
dev
stage
prod
```

Network flow:

```text
Internet
 ↓
WAF
 ↓
ALB
 ↓
Private ECS
 ↓
PrivateLink
 ↓
Bedrock
```

---

# 24. CI/CD

Application deployment:

```text
Git Push
 ↓
Unit Test
 ↓
Integration Test
 ↓
Security Scan
 ↓
Build
 ↓
ECR
 ↓
Stage
 ↓
Load / Evaluation Gate
 ↓
ECS Blue-Green / Canary
 ↓
Production
```

Infrastructure deployment:

```text
Terraform fmt
 ↓
Terraform validate
 ↓
Terraform plan
 ↓
Approval
 ↓
Terraform apply
```

Model and prompt release:

```text
Evaluation
 ↓
Certification
 ↓
Canary
 ↓
Promotion
```

---

# 25. Repository Structure

```text
enterprise-ai-platform/

├── services/
│   ├── gateway/
│   │   ├── api/
│   │   ├── auth/
│   │   ├── tenant/
│   │   ├── policy/
│   │   ├── quota/
│   │   ├── guardrails/
│   │   ├── cache/
│   │   ├── routing/
│   │   ├── inference/
│   │   ├── streaming/
│   │   └── telemetry/
│   │
│   ├── worker/
│   └── admin/
│
├── libs/
│   ├── schemas/
│   ├── policies/
│   ├── models/
│   └── telemetry/
│
├── infra/
│   ├── modules/
│   └── environments/
│       ├── dev/
│       ├── stage/
│       └── prod/
│
├── evals/
│   ├── datasets/
│   ├── runners/
│   └── reports/
│
├── loadtests/
│   ├── bedrock/
│   ├── guardrails/
│   ├── tenant/
│   └── failure/
│
├── dashboards/
├── policies/
├── docs/
└── .github/workflows/
```

---

# 26. Build Roadmap

| Milestone                    | Scope                                                                               | Exit Criteria                           |
| ---------------------------- | ----------------------------------------------------------------------------------- | --------------------------------------- |
| **M0 — Walking Skeleton**    | ECS + `/chat` + Bedrock + minimal telemetry                                         | End-to-end inference works              |
| **M1 — Identity**            | OIDC/JWT + RBAC/ABAC + tenant resolution                                            | Tenant identity is trustworthy          |
| **M2 — Policy Plane**        | Tenant config + policy epoch + kill switch + push invalidation + bounded TTL        | Policy changes have bounded propagation |
| **M3 — Safety**              | Input/output guardrails + fail-closed behavior                                      | Safety invariant holds                  |
| **M4 — Gateway Reliability** | Cache + retry + jitter + circuit breaker + certified fallback + stream cancellation | Failure paths behave correctly          |
| **M5 — Observability**       | OTel + latency/tokens/cost/SLO + PII-safe logging                                   | Every request is traceable              |
| **M6 — Load / Chaos**        | Bedrock quota tests + 429 + noisy neighbor + policy/failure injection               | SLO and invariants survive load         |
| **V1 Release**               | M0–M6                                                                               | Enterprise MVP                          |
| **M7 — Async**               | SQS + workers + Step Functions + Batch                                              | Long-running workloads supported        |
| **M8 — FinOps**              | Budgets + chargeback/showback + anomaly alerts                                      | Cost governance operational             |
| **M9 — Model Lifecycle**     | Golden dataset + certification + canary + rollback                                  | Controlled model deployment             |
| **M10 — Portal**             | Self-service onboarding + approval workflows                                        | Full enterprise platform                |

---

# 27. V1 Definition

`/v1/chat` alone is not V1.

It is only the M0 walking skeleton.

The real Enterprise V1 includes:

```text
OIDC / JWT
     ↓
AuthN / AuthZ
     ↓
Tenant Resolution
     ↓
Emergency Kill Switch
     ↓
Versioned Policy
     ↓
Quota / Rate Limit / Budget
     ↓
Input Guardrail
     ↓
Policy-Aware Cache
     ↓
Certified Model Router
     ↓
Retry / Circuit Breaker / Fallback
     ↓
Bedrock
     ↓
Output Guardrail
     ↓
SSE Response

        +
Telemetry
Audit
Cost
Load Testing
Failure Testing
```

---

# 28. V1 Acceptance Criteria

V1 should demonstrate that:

```text
Tenant A cannot escape tenant isolation.

Guardrail failure cannot silently bypass safety.

Policy revocation takes effect within a bounded time.

Cache cannot bypass a newer security or guardrail policy.

Fallback cannot bypass model evaluation.

Client disconnect propagates upstream cancellation.

429 throttling does not create retry storms.

Control-plane outage does not automatically stop the data plane.

Production traffic only reaches certified models.

Telemetry does not expose raw prompts or responses by default.
```

At this point, the system is no longer a Bedrock API demo.

It is a production-oriented **Enterprise GenAI Platform / LLM Gateway** with explicit handling for multi-tenancy, policy consistency, safety failure modes, reliability, capacity, governance, and model lifecycle.
