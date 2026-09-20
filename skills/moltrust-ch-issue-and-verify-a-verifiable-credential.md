---
generated: '2026-09-19'
method: generated
name: Issue and verify a W3C Verifiable Credential
description: Issue a W3C VC for a registered agent DID, verify it (signature, expiry, revocation), and verify a
  full AAE delegation chain.
api: openapi/moltrust-ch-openapi.yml
operations:
- issue_vc_credentials_issue_post
- verify_vc_credentials_verify_post
- verify_delegation_chain_endpoint_credentials_verify_chain_post
- revocation_status_identity_revocation_status__did__get
source: Grounded in openapi/moltrust-ch-openapi.yml (captured 2026-09-19 from https://api.moltrust.ch/openapi.json;
  verbatim copy in openapi/_original/). Every operationId verified verbatim by the generating script. Conventions
  per conventions/, errors per errors/, limits per rate-limits/, pricing per plans/.
---

# Issue and verify a W3C Verifiable Credential

Issue a W3C VC for a registered agent DID, verify it (signature, expiry, revocation), and verify a full AAE delegation chain.

## Steps

1. **Issue** - `issue_vc_credentials_issue_post` (`POST /credentials/issue`, `X-API-Key`, body `IssueVCRequest {subject_did, credential_type}`). 2 credits. The issuer is `did:web:api.moltrust.ch`; its keys are published at `https://api.moltrust.ch/.well-known/jwks.json` and in `/.well-known/did.json`.
2. **Verify** - `verify_vc_credentials_verify_post` (`POST /credentials/verify`, body `VerifyVCRequest {credential: {...}}`). 2 credits. The response reports `signature`, `expiration`, `revocation` and `delegation_chain` checks (Trust Registry Binding 3.3).
3. **Verify a chain** - `verify_delegation_chain_endpoint_credentials_verify_chain_post` (`POST /credentials/verify-chain`, body `DelegationChainRequest {credential_chain: [...]}`) walks an AAE delegation chain end to end.
4. **Check revocation independently** - `revocation_status_identity_revocation_status__did__get` (`GET /identity/revocation-status/{did}`) before trusting a cached credential; `POST /identity/revoke/{did}` cascades up to 8 hops and emits CAEP events (see the CAEP skill).
5. **Offline path.** The provider ships `@moltrust/verify` (npm) for verifying a VC against Base L2 with no API key; prefer it when the API is unreachable, since a verification failure must fail closed.

## Rules an agent must follow

- **Auth.** Send your key in the `X-API-Key` header (free key: `POST /auth/signup` with an email, or `POST /auth/signup-did` by Ed25519 proof of possession - no email, no card). Trust-gated reads accept `X-MolTrust-DID` instead. See `../authentication/moltrust-ch-authentication.yml`.
- **No idempotency key exists anywhere in the 186-operation spec** (`../conventions/moltrust-ch-conventions.yml`, `idempotency.coverage: none`). A timed-out POST that mints something (a registration, a credential, a delegation, a transfer) must be re-read before it is retried, never blind-retried.
- **Credits are spent per call and purchases are final.** `GET /credits/pricing` is the machine-readable cost table (`../plans/moltrust-ch-credits-pricing.json`); most writes cost 1-2 credits, reads like health/resolve/pricing cost 0. The Terms say credit purchases are non-refundable, so a duplicate write is real money.
- **Errors are FastAPI-shaped, not RFC 9457**: `{"detail": ...}` where `detail` is a string (401/400/404) or a `ValidationError[]` array (422). See `../errors/moltrust-ch-problem-types.yml`.
- **Rate limits carry no headers.** Free tier is 60 calls/hour (pricing page) and a 429 arrives with a JSON body and no `Retry-After`. Back off exponentially. See `../rate-limits/moltrust-ch-rate-limits.yml`.
- **A fresh agent's trust score is `null`, not 0**, until it has three endorsements - treat `null` as "unscored", not as a failure.
