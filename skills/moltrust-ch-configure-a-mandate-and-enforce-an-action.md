---
generated: '2026-09-19'
method: generated
name: Configure an AAE mandate and enforce an action against it
description: Configure delegation permissions for an agent, mint and verify a UCAN 0.10.0 delegation, then run a
  stateless enforce/check before acting and ratify a DENY/PENDING record.
api: openapi/moltrust-ch-openapi.yml
operations:
- configure_delegation_delegation_configure_post
- delegation_create_delegation_create_post
- delegation_verify_delegation_verify_post
- get_delegations_identity_delegations__did__get
- enforce_check_endpoint_enforce_check_post
- enforce_ratify_endpoint_enforce_ratify_post
source: Grounded in openapi/moltrust-ch-openapi.yml (captured 2026-09-19 from https://api.moltrust.ch/openapi.json;
  verbatim copy in openapi/_original/). Every operationId verified verbatim by the generating script. Conventions
  per conventions/, errors per errors/, limits per rate-limits/, pricing per plans/.
---

# Configure an AAE mandate and enforce an action against it

Configure delegation permissions for an agent, mint and verify a UCAN 0.10.0 delegation, then run a stateless enforce/check before acting and ratify a DENY/PENDING record.

## Steps

1. **Configure** - `configure_delegation_delegation_configure_post` (`POST /delegation/configure`, `X-API-Key`, admin or agent owner). This is where the AAE MANDATE / CONSTRAINTS / VALIDITY blocks are set; the credential from `/identity/register` does not embed them (developers page, "AAE").
2. **Mint a delegation** - `delegation_create_delegation_create_post` (`POST /delegation/create`, body `DelegationCreateRequest {delegator_did, audience_did, capabilities, ttl_seconds?, proofs?}`). Returns a UCAN 0.10.0 JWT bounded by the delegator's configuration. 2 credits.
3. **Verify it** - `delegation_verify_delegation_verify_post` (`POST /delegation/verify`, body `DelegationVerifyRequest {token, expected_audience?}`). 1 credit. `get_delegations_identity_delegations__did__get` (`GET /identity/delegations/{did}`) lists the chain for inspection.
4. **Check before acting** - `enforce_check_endpoint_enforce_check_post` (`POST /enforce/check`, headers `X-API-Key` + `X-MolTrust-DID`). The verdict is PERMIT / DENY / PENDING and depends only on `mandate + transaction` - no server state - so recompute it locally with `moltrust-enforce` (`pip install moltrust-enforce`) and refuse to act if the two disagree. An unreachable server is a DENY.
5. **Ratify** - `enforce_ratify_endpoint_enforce_ratify_post` (`POST /enforce/ratify`) appends a second record referencing the DENY/PENDING by `core_digest`; the original is never modified (append-only history).

## Rules an agent must follow

- **Auth.** Send your key in the `X-API-Key` header (free key: `POST /auth/signup` with an email, or `POST /auth/signup-did` by Ed25519 proof of possession - no email, no card). Trust-gated reads accept `X-MolTrust-DID` instead. See `../authentication/moltrust-ch-authentication.yml`.
- **No idempotency key exists anywhere in the 186-operation spec** (`../conventions/moltrust-ch-conventions.yml`, `idempotency.coverage: none`). A timed-out POST that mints something (a registration, a credential, a delegation, a transfer) must be re-read before it is retried, never blind-retried.
- **Credits are spent per call and purchases are final.** `GET /credits/pricing` is the machine-readable cost table (`../plans/moltrust-ch-credits-pricing.json`); most writes cost 1-2 credits, reads like health/resolve/pricing cost 0. The Terms say credit purchases are non-refundable, so a duplicate write is real money.
- **Errors are FastAPI-shaped, not RFC 9457**: `{"detail": ...}` where `detail` is a string (401/400/404) or a `ValidationError[]` array (422). See `../errors/moltrust-ch-problem-types.yml`.
- **Rate limits carry no headers.** Free tier is 60 calls/hour (pricing page) and a 429 arrives with a JSON body and no `Retry-After`. Back off exponentially. See `../rate-limits/moltrust-ch-rate-limits.yml`.
- **A fresh agent's trust score is `null`, not 0**, until it has three endorsements - treat `null` as "unscored", not as a failure.
