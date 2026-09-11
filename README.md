# bedrock-gateway-platform

Multi-tenant Enterprise LLM Gateway on AWS Bedrock. Building toward the
V1 Enterprise MVP (M0–M6) in `docs/ROADMAP.md`. Currently: **M0**
(`POST /v1/chat` → Bedrock Converse API → response, structured JSON
telemetry), **M1** (OIDC/JWT auth, tenant identity derived only from
verified token claims, RBAC), and **M2** (tenant policy plane: state/kill
switch, policy epoch, push-invalidated + bounded-TTL policy cache,
per-tenant rate limit, model allowlist, admin state API).

## Quickstart (local)

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt   # -dev pulls in httpx for tests too
cp .env.example .env                  # adjust if needed; defaults are fine for local dev
export $(grep -v '^#' .env | xargs)   # or use direnv/dotenv-cli

# AWS credentials come from your normal chain (env vars / ~/.aws/credentials
# / SSO / instance role) — not from .env. You need `bedrock:InvokeModel`
# (or equivalent Converse permission) on the target model in BEDROCK_MODEL_ID,
# and model access enabled for that model in the Bedrock console for your region.

python -m services.gateway.main
# -> gateway-api listening on http://0.0.0.0:8080
```

No real OIDC provider is configured by default (`OIDC_JWKS_URL` is
empty), so the server verifies tokens against a local dev keypair it
generates on first run (`.dev/jwt_keypair.json`, gitignored). Mint a
matching token with `scripts/generate_dev_token.py`:

```bash
curl -s http://localhost:8080/healthz   # unauthenticated liveness probe

# tenant_id must be one already configured in policies/tenants.yaml
# (finance / search / sandbox out of the box) -- see M2 below.
TOKEN=$(python scripts/generate_dev_token.py -q --tenant-id finance --roles developer)

curl -s -X POST http://localhost:8080/v1/chat \
  -H "authorization: Bearer $TOKEN" \
  -H 'content-type: application/json' \
  -d '{"messages":[{"role":"user","content":"Say hello in one sentence."}]}'
```

To verify against a real IdP instead, set `OIDC_JWKS_URL`, `OIDC_ISSUER`,
and `OIDC_AUDIENCE` and the server switches to `JwksVerifier` automatically
(see `services/gateway/auth/jwt_verifier.py`).

Tenant policy lives in `policies/tenants.yaml` (state, model allowlist,
rate limit, guardrail policy, route set — see M2 below). To flip a
tenant's state at runtime (the kill switch) without restarting the
server:

```bash
ADMIN_TOKEN=$(python scripts/generate_dev_token.py -q --tenant-id platform --roles platform_admin)

curl -s -X PUT http://localhost:8080/v1/admin/tenants/finance/state \
  -H "authorization: Bearer $ADMIN_TOKEN" \
  -H 'content-type: application/json' \
  -d '{"state": "SUSPENDED"}'
```

## Tests

No real AWS calls, no network — a fake `ConverseClient` stands in for
Bedrock (see `services/gateway/tests/fakes.py`), tests mint their own
locally-signed JWTs (see `services/gateway/tests/auth_fixtures.py`), and
policy/rate-limit tests use a `FakeClock` (see
`services/gateway/tests/fake_clock.py`) instead of real `time.sleep()`.

```bash
python -m unittest discover -s services/gateway/tests -t .
```

46 tests: M0/M1 coverage plus M2 — kill switch blocks a
SUSPENDED/EMERGENCY_BLOCK tenant before Bedrock is ever called, an
unprovisioned tenant is rejected the same way, per-tenant rate limiting
(and that tenant A's burst never throttles tenant B), the policy TTL
bound holds even without a push, and an admin state change propagates to
the very next request instead of waiting out the TTL.

## Docker

```bash
docker build -t gateway-api:m0 .
docker run --rm -p 8080:8080 \
  -e AWS_REGION=us-east-1 \
  -e AWS_ACCESS_KEY_ID=... -e AWS_SECRET_ACCESS_KEY=... \
  gateway-api:m0
```

Intended to run as an ECS Fargate task (section 2 of the plan) behind an
ALB + WAF, reaching Bedrock over a VPC PrivateLink endpoint — none of that
infra is provisioned yet (`infra/` is a placeholder for the Terraform
modules in M2+).

## Layout

```
services/gateway/
  api/          # request/response schemas + route handlers (routes.py public, admin_routes.py admin)
  auth/         # JWT verification, identity, RBAC (M1)
  policy/       # tenant policy model/store/cache, rate limiter (M2)
  inference/    # Bedrock Converse client (retry/backoff, no boto3 at import time)
  telemetry/    # structured JSON logging + request_id/duration_ms middleware
  tests/        # unit tests + fakes/fixtures (no AWS, no network needed)
  config.py     # env -> Settings (the only module that reads os.environ)
  main.py       # app factory / entrypoint
  pipeline.py   # request pipeline stages (auth + policy so far; guardrails/cache/router land in M3-M4)
scripts/
  generate_dev_token.py   # mint a local dev JWT for curl-testing
policies/
  tenants.yaml  # tenant policy config (stands in for the DynamoDB table in the full plan)
docs/
  ROADMAP.md        # milestone status
  DESIGN-NOTES.md   # deviations from the plan and why
```

## What's next (M3)

Input/output guardrails with fail-closed behavior: a guardrail
timeout/error on a STRICT or STANDARD safety class must reject the
request rather than let it reach the model — see `docs/ROADMAP.md`.
