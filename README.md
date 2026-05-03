# LIQAA · Cross-SDK compliance suite

> **Proof that all four LIQAA SDKs (JS, PHP, Python, Go) implement identical behaviour.**

A single test scenario, executed in matrix CI against every SDK release, every commit. If any SDK drifts from the spec, this turns red. Inspired by [`go/x/text`](https://github.com/golang/text), [`oauth2-test`](https://github.com/oauth2/oauth2-test), and Stripe's internal test suite (which they don't publish).

[![compliance](https://github.com/hartemyaakoub/liqaa-compliance/actions/workflows/matrix.yml/badge.svg)](https://github.com/hartemyaakoub/liqaa-compliance/actions/workflows/matrix.yml)

## The scenario

A real customer flow, executed step-by-step:

1. **Init client** with `sk_live_test_…`
2. **Create a room** named `compliance_<random>`
3. **Issue a JWT** for user `agent@compliance.test`, scoped to that room, TTL 60s
4. **Decode the JWT**, assert claims match: `sub`, `room`, `exp`, `iss`, `iat`
5. **Register a webhook** subscription for `call.ended`
6. **Fire a synthetic webhook delivery** with a known payload
7. **Verify the signature** server-side (constant time)
8. **Reject a tampered signature** with the SDK's verifier
9. **Reject an expired timestamp** (older than 5 min)
10. **List rooms**, assert pagination metadata matches the OpenAPI spec
11. **End the room**, assert it disappears from the list
12. **Delete the webhook**, assert it disappears
13. **Idempotency:** repeat step 2 with the same `Idempotency-Key`, assert same room ID returned
14. **Rate limit:** burst 30 token issuances in 1s, assert exactly 20 succeed (we publish 20/s)
15. **Error shapes:** intentionally pass invalid input, assert the SDK throws / returns the documented error type

15 assertions × 4 SDKs = **60 cells** in the matrix. All must be green.

## Why this matters

Polyrepo SDKs (one per language, see [ADR-007](https://github.com/hartemyaakoub/liqaa-architecture/blob/main/adrs/007-monorepo-vs-polyrepo-sdks.md)) have one famous failure mode: behaviour drift. The PHP SDK fixes a bug, the Python SDK forgets to. Six months later, a customer ports their app from PHP to Python and `dropAfterTimeout()` no longer fires.

The OpenAPI spec ([`liqaa-openapi`](https://github.com/hartemyaakoub/liqaa-openapi)) prevents *type* drift. This suite prevents *behaviour* drift.

## Architecture

```
.
├─ scenarios/
│  ├─ 01-rooms.yaml          · declarative scenario steps
│  ├─ 02-tokens.yaml
│  ├─ 03-webhooks.yaml
│  ├─ 04-idempotency.yaml
│  └─ 05-error-shapes.yaml
│
├─ runners/
│  ├─ js/                    · Node 18/20/22 → liqaa-js
│  ├─ php/                   · PHP 8.2/8.3/8.4 → liqaa-php
│  ├─ python/                · Python 3.10/3.11/3.12/3.13 → liqaa-python
│  └─ go/                    · Go 1.21/1.22/1.23 → liqaa-go
│
└─ scripts/
   ├─ run-all.sh             · one command runs the matrix locally
   └─ generate-matrix.mjs    · emits GitHub Actions matrix from scenarios
```

Each runner is a thin shim: it reads a scenario YAML, calls the SDK methods, asserts the responses. The shims share zero code — that's the point. If any runner cheats, real behaviour will diverge from what's tested.

## Running locally

```bash
git clone https://github.com/hartemyaakoub/liqaa-compliance
cd liqaa-compliance
export LIQAA_TEST_SK=sk_test_…   # get one at https://liqaa.io/console (test mode)
./scripts/run-all.sh
```

You'll need: Node 20+, PHP 8.3+, Python 3.11+, Go 1.22+. Or use the Codespaces config (cmd-shift-P → "Codespaces: Create new").

## CI

The [matrix.yml](./.github/workflows/matrix.yml) workflow runs on:

- Every push to `main` of this repo
- Every release tag of `liqaa-js`, `liqaa-php`, `liqaa-python`, `liqaa-go` (via `repository_dispatch`)
- Daily at 03:00 UTC (catches downstream dep changes)

Total runtime: ~6 min. Cost: $0 (free GitHub Actions minutes for public repos).

## Adding a new scenario

1. Drop a YAML file in `scenarios/`
2. The runners auto-discover it
3. Open a PR — CI runs the new scenario across all 4 SDKs

If three SDKs pass and one fails, **the SDK is wrong**, not the scenario.

## License

[Apache 2.0](./LICENSE) — fork it, run it against your own video API, learn from it.
