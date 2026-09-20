---
generated: '2026-09-19'
method: generated
name: Register an agent and read its trust score
description: Mint a free API key, register a W3C DID + signed AgentTrustCredential for an agent, verify the DID
  resolves, then read the public trust score and its signed breakdown.
api: openapi/moltrust-ch-openapi.yml
operations:
- signup_for_api_key_auth_signup_post
- register_agent_identity_register_post
- verify_agent_identity_verify__did__get
- resolve_did_identity_resolve__did__get
- get_trust_score_skill_trust_score__did__get
- get_agent_class_identity_agent_type__did__get
source: Grounded in openapi/moltrust-ch-openapi.yml (captured 2026-09-19 from https://api.moltrust.ch/openapi.json;
  verbatim copy in openapi/_original/). Every operationId verified verbatim by the generating script. Conventions
  per conventions/, errors per errors/, limits per rate-limits/, pricing per plans/.
---

# Register an agent and read its trust score

Mint a free API key, register a W3C DID + signed AgentTrustCredential for an agent, verify the DID resolves, then read the public trust score and its signed breakdown.

## Steps

1. **Get a key** - `signup_for_api_key_auth_signup_post` (`POST /auth/signup`, body `{"email": "..."}`). One key per email (Terms 3). The response carries `api_key`; export it as `MOLTRUST_API_KEY`. Cost: 0 credits.
2. **Register the agent** - `register_agent_identity_register_post` (`POST /identity/register`, header `X-API-Key`, body `RegisterRequest {display_name, platform, email?, erc8004?}`). Returns the agent's `did:moltrust:...`, a signed W3C Verifiable Credential and 100 credits. Costs 1 credit per `/credits/pricing` but is documented as free on registration. Set `erc8004: true` only if you want the dual on-chain registration on Base.
3. **Prove it resolves** - `resolve_did_identity_resolve__did__get` (`GET /identity/resolve/{did}`, 0 credits) returns the DID Document; `verify_agent_identity_verify__did__get` (`GET /identity/verify/{did}`, 1 credit) returns the verification status. The same DID resolves through the DIF Universal Resolver at `https://uresolver.moltrust.ch/1.0/identifiers/{did}`.
4. **Read the trust score** - `get_trust_score_skill_trust_score__did__get` (`GET /skill/trust-score/{did}`, public, 1h cache). Expect `trust_score: null`, `withheld: true` until three endorsements exist. The payload carries `registry_signature` (Ed25519 over the JCS-canonical `{did, trust_score, computed_at, valid_until, policy_version}`) - verify it against `GET /.well-known/registry-key.json` if you cache the score.
5. **Read the governance tier** - `get_agent_class_identity_agent_type__did__get` (`GET /identity/agent-type/{did}`) returns `agent_class` (orchestrator / autonomous / human_initiated / copilot) and the review cadence attached to it.

## Rules an agent must follow

- **Auth.** Send your key in the `X-API-Key` header (free key: `POST /auth/signup` with an email, or `POST /auth/signup-did` by Ed25519 proof of possession - no email, no card). Trust-gated reads accept `X-MolTrust-DID` instead. See `../authentication/moltrust-ch-authentication.yml`.
- **No idempotency key exists anywhere in the 186-operation spec** (`../conventions/moltrust-ch-conventions.yml`, `idempotency.coverage: none`). A timed-out POST that mints something (a registration, a credential, a delegation, a transfer) must be re-read before it is retried, never blind-retried.
- **Credits are spent per call and purchases are final.** `GET /credits/pricing` is the machine-readable cost table (`../plans/moltrust-ch-credits-pricing.json`); most writes cost 1-2 credits, reads like health/resolve/pricing cost 0. The Terms say credit purchases are non-refundable, so a duplicate write is real money.
- **Errors are FastAPI-shaped, not RFC 9457**: `{"detail": ...}` where `detail` is a string (401/400/404) or a `ValidationError[]` array (422). See `../errors/moltrust-ch-problem-types.yml`.
- **Rate limits carry no headers.** Free tier is 60 calls/hour (pricing page) and a 429 arrives with a JSON body and no `Retry-After`. Back off exponentially. See `../rate-limits/moltrust-ch-rate-limits.yml`.
- **A fresh agent's trust score is `null`, not 0**, until it has three endorsements - treat `null` as "unscored", not as a failure.
