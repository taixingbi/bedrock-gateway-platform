# bedrock-gateway-platform

Multi-tenant Enterprise LLM Gateway on AWS Bedrock. **V1 (M0–M6) is
complete** — see `docs/ROADMAP.md` for the full milestone-by-milestone
breakdown and the plan's acceptance-criteria checklist:

- **M0** — `POST /v1/chat` → Bedrock Converse API → response, structured JSON telemetry
- **M1** — OIDC/JWT auth; tenant identity derived only from verified token claims; RBAC
- **M2** — tenant policy plane: state/kill switch, policy epoch, push-invalidated + bounded-TTL policy cache, per-tenant rate limit, model allowlist, admin state API
- **M3** — input/output guardrails with fail-closed behavior on timeout/error
- **M4** — policy-aware response cache, per-model circuit breaker, certified fallback routing, SSE streaming with client-disconnect cancellation
- **M5** — OpenTelemetry tracing, per-request cost estimate and SLO breach flag, opt-in redacted debug capture kept separate from operational telemetry
- **M6** — load/chaos scenarios (circuit breaker under load, tenant isolation, kill-switch/policy-update propagation, guardrail fail-closed) — all automated and CI-runnable, see `docs/LOAD_TESTING.md`

M7 (Async), M8 (FinOps), M9 (Model Lifecycle), and M10 (Portal) are the
rest of the platform in `plan.md` and are not started.

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

For an SSE stream instead of a single JSON response, pass `"stream": true`:

```bash
curl -N -s -X POST http://localhost:8080/v1/chat \
  -H "authorization: Bearer $TOKEN" \
  -H 'content-type: application/json' \
  -d '{"stream": true, "messages":[{"role":"user","content":"Count to five."}]}'
```

## Tests

No real AWS calls, no network — a fake `ConverseClient` stands in for
Bedrock (see `services/gateway/tests/fakes.py`), tests mint their own
locally-signed JWTs (see `services/gateway/tests/auth_fixtures.py`),
policy/rate-limit tests use a `FakeClock` (see
`services/gateway/tests/fake_clock.py`) instead of real `time.sleep()`,
and guardrail failure/timeout scenarios use a `FakeGuardrailClient` (see
`services/gateway/tests/fake_guardrail.py`) instead of racing a real
clock against a real deadline.

```bash
python -m unittest discover -s services/gateway/tests -t .
```

113 tests: M0-M4 coverage plus M5 — a chat span carries the full
telemetry attribute set (checked against an in-memory OTel exporter, not
parsed out of console JSON), cost/SLO-breach calculations are correct,
PII redaction round-trips through `DebugCaptureStore`, an opt-in tenant's
interaction is actually captured while a default tenant's is not, and a
grep-the-logs test proves a request's raw message content never appears
in any operational JSON log line.

Note: `python -m unittest` runs quietly by design -- `services/gateway/tests/__init__.py`
installs a no-op global tracer before any test imports `create_app()`, so
you won't see the console span exporter's output during the suite. Run
the server for real (`python -m services.gateway.main`) to see it.

## Load / chaos testing (M6)

```bash
python -m unittest discover -s loadtests -t .
```

5 scenarios, each firing a burst of concurrent requests at the app over
`httpx.ASGITransport` (no live server needed) with fault-injecting fakes
distinct from the unit-test ones (`loadtests/fault_injection.py` — these
inject faults across *many* concurrent calls with thread-safe counters):
circuit breaker stops a retry storm under throttling, tenant noisy-
neighbor isolation holds under simultaneous bursts, kill-switch blocks
the very next request after an admin flip, a policy_epoch bump
invalidates the cache under concurrent read/write races, and a STRICT
tenant fails closed for every one of many concurrent requests when its
guardrail is down.

Each scenario also has a human-run `locustfile.py` for real HTTP
throughput/latency against a live gateway process — including,
deliberately by hand, against real Bedrock if you want an actual quota
number (costs money, needs your AWS credentials, never run
automatically). Full instructions: `docs/LOAD_TESTING.md`.

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
  guardrails/   # GuardrailClient seam, basic regex impl, fail-closed enforcement (M3)
  cache/        # policy-aware response cache: key derivation + in-memory store (M4)
  routing/      # circuit breaker + certified router with fallback (M4)
  inference/    # Bedrock Converse client (retry/backoff + streaming, no boto3 at import time)
  telemetry/    # structured JSON logging + middleware, OTel tracing, cost/SLO, debug capture (M5)
  tests/        # unit tests + fakes/fixtures (no AWS, no network needed)
  config.py     # env -> Settings (the only module that reads os.environ)
  main.py       # app factory / entrypoint
  pipeline.py   # request pipeline stages (auth/policy/guardrails)
  streaming.py  # SSE + client-disconnect cancellation (M4)
scripts/
  generate_dev_token.py   # mint a local dev JWT for curl-testing
policies/
  tenants.yaml     # tenant policy config (stands in for the DynamoDB table in the full plan)
  route_sets.yaml  # certified model route sets: primary + fallbacks (M4)
loadtests/
  fault_injection.py       # multi-call fault-injecting fakes (M6)
  harness.py                # ASGI-direct concurrency test harness (M6)
  bedrock/ tenant/ failure/ guardrails/   # test_scenario.py (automated) + locustfile.py (manual) per area
docs/
  ROADMAP.md         # milestone status
  DESIGN-NOTES.md    # deviations from the plan and why
  LOAD_TESTING.md    # M6 scenarios: what they prove and how to run them
```

## What's next (M7+)

V1 (M0–M6) is complete. The next milestones in `plan.md` are M7 (Async:
SQS + workers + Step Functions + Batch for long-running workloads), M8
(FinOps: real budget tracking, chargeback/showback, anomaly alerts), M9
(Model Lifecycle: golden-dataset evaluation, certification, canary
rollout), and M10 (Portal: self-service onboarding + approval
workflows) — none started yet.
