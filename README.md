# bedrock-gateway-platform — M0 walking skeleton

Multi-tenant Enterprise LLM Gateway on AWS Bedrock. This is **M0** of the
build roadmap (see `docs/ROADMAP.md`): the minimal end-to-end slice —
`POST /v1/chat` → Bedrock Converse API → response, with structured JSON
telemetry — that everything else (identity, policy plane, guardrails,
caching, certified routing, streaming, async jobs, FinOps, model
lifecycle, self-service portal) gets layered onto in later milestones.

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

```bash
curl -s http://localhost:8080/healthz

curl -s -X POST http://localhost:8080/v1/chat \
  -H 'content-type: application/json' \
  -d '{"messages":[{"role":"user","content":"Say hello in one sentence."}]}'
```

## Tests

No real AWS calls, no network — a fake `ConverseClient` stands in for
Bedrock (see `services/gateway/tests/fakes.py`).

```bash
python -m unittest discover -s services/gateway/tests -t .
```

11 tests: health check + request-id propagation, success path, model
override / default, request validation (empty messages, last message must
be `user`, invalid JSON), and upstream-error mapping (`ThrottlingException`
→ 429, unrecognized error → 502).

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
  api/          # request/response schemas + route handlers
  inference/    # Bedrock Converse client (retry/backoff, no boto3 at import time)
  telemetry/    # structured JSON logging + request_id/duration_ms middleware
  tests/        # unit tests + fakes (no AWS needed)
  config.py     # env -> Settings (the only module that reads os.environ)
  main.py       # app factory / entrypoint
docs/
  ROADMAP.md        # milestone status
  DESIGN-NOTES.md   # deviations from the plan and why
```

## What's next (M1)

OIDC/JWT verification, tenant_id resolution from the authenticated
identity (never a trusted header), and RBAC/ABAC — see `docs/ROADMAP.md`.
# bedrock-gateway-platform
