---
generated: '2026-09-19'
method: generated
name: Pay-per-call wallet trust check with x402
description: 'Score a Base wallet through MoltGuard: try the free rate-limited variant, then pay the x402 402 challenge
  in USDC on Base for the full score, detail and sybil scan.'
api: openapi/moltrust-ch-moltguard-openapi.yml
operations:
- getAgentScoreFree
- getAgentScorePaid
- getAgentDetailPaid
- getSybilScan
- getApiInfo
source: Grounded in openapi/moltrust-ch-moltguard-openapi.yml (captured 2026-09-19 from https://api.moltrust.ch/guard/openapi.json;
  verbatim copy in openapi/_original/). Every operationId verified verbatim by the generating script. Conventions
  per conventions/, errors per errors/, limits per rate-limits/, pricing per plans/.
---

# Pay-per-call wallet trust check with x402

Score a Base wallet through MoltGuard: try the free rate-limited variant, then pay the x402 402 challenge in USDC on Base for the full score, detail and sybil scan.

## Steps

1. **Read the price list first** - `getApiInfo` (`GET /guard/api/info`, free) or the discovery document `https://api.moltrust.ch/.well-known/x402.json` (`../well-known/moltrust-ch-x402.json`). Prices are per call in USDC on Base (eip155:8453): score $0.05, detail $0.05, sybil scan $0.10.
2. **Free path** - `getAgentScoreFree` (`GET /guard/api/agent/score-free/{address}`). Rate-limited to 1 request per 10 minutes; a 429 returns `RateLimitError {message}` and no headers.
3. **Paid path** - `getAgentScorePaid` (`GET /guard/api/agent/score/{address}`). Without an `X-PAYMENT` header the response is `402` with a `PaymentRequired` body (`x402.accepts[]` names scheme, network, `maxAmountRequired` in USDC base units and `payTo`). Pay, retry with `X-PAYMENT: x402 <base64 receipt>`.
4. **Go deeper** - `getAgentDetailPaid` (`GET /guard/api/agent/detail/{address}`) for the breakdown, `getSybilScan` (`GET /guard/api/sybil/scan/{address}`) for cluster indicators. Each is a separate payment; every `x-moltrust-pricing` extension on the operation states the amount.
5. **Budget guard.** A 402 is not an error to retry; it is a quote. Enforce a spend cap client-side - the provider's own spend cap (`PUT /operators/{operator_did}/agents/{agent_did}/budget-cap`) covers credits, not x402 wallet spend.

## Rules an agent must follow

- **Auth.** Send your key in the `X-API-Key` header (free key: `POST /auth/signup` with an email, or `POST /auth/signup-did` by Ed25519 proof of possession - no email, no card). Trust-gated reads accept `X-MolTrust-DID` instead. See `../authentication/moltrust-ch-authentication.yml`.
- **No idempotency key exists anywhere in the 186-operation spec** (`../conventions/moltrust-ch-conventions.yml`, `idempotency.coverage: none`). A timed-out POST that mints something (a registration, a credential, a delegation, a transfer) must be re-read before it is retried, never blind-retried.
- **Credits are spent per call and purchases are final.** `GET /credits/pricing` is the machine-readable cost table (`../plans/moltrust-ch-credits-pricing.json`); most writes cost 1-2 credits, reads like health/resolve/pricing cost 0. The Terms say credit purchases are non-refundable, so a duplicate write is real money.
- **Errors are FastAPI-shaped, not RFC 9457**: `{"detail": ...}` where `detail` is a string (401/400/404) or a `ValidationError[]` array (422). See `../errors/moltrust-ch-problem-types.yml`.
- **Rate limits carry no headers.** Free tier is 60 calls/hour (pricing page) and a 429 arrives with a JSON body and no `Retry-After`. Back off exponentially. See `../rate-limits/moltrust-ch-rate-limits.yml`.
- **A fresh agent's trust score is `null`, not 0**, until it has three endorsements - treat `null` as "unscored", not as a failure.
