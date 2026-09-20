---
generated: '2026-09-19'
method: generated
name: Poll CAEP trust-change events and verify signed scores
description: Subscribe to trust changes for a DID by polling the MolTrust CAEP Profile v1 channel, acknowledge events,
  and verify the Ed25519 registry signature on cached trust scores.
api: openapi/moltrust-ch-openapi.yml
operations:
- caep_pending_caep_pending__did__get
- caep_acknowledge_caep_acknowledge__event_id__post
- well_known_registry_key__well_known_registry_key_json_get
- get_trust_score_skill_trust_score__did__get
source: Grounded in openapi/moltrust-ch-openapi.yml (captured 2026-09-19 from https://api.moltrust.ch/openapi.json;
  verbatim copy in openapi/_original/). Every operationId verified verbatim by the generating script. Conventions
  per conventions/, errors per errors/, limits per rate-limits/, pricing per plans/.
---

# Poll CAEP trust-change events and verify signed scores

Subscribe to trust changes for a DID by polling the MolTrust CAEP Profile v1 channel, acknowledge events, and verify the Ed25519 registry signature on cached trust scores.

## Steps

1. **Fetch the registry key once** - `well_known_registry_key__well_known_registry_key_json_get` (`GET /.well-known/registry-key.json`, `Cache-Control: max-age=3600`). kid `moltrust-registry-2026-v1`, Ed25519 JWK; the same key is in `/.well-known/jwks.json`.
2. **Poll** - `caep_pending_caep_pending__did__get` (`GET /caep/pending/{did}?limit=50&since=<last event_id>`). Default poll interval 30 s; limit 120/h per DID (docs/caep.html). Event types: `trust_score_change` (delta >= 10 points, live), `flag_added`, `flag_removed`, `did_revoked` (schema present, emitters staged).
3. **Acknowledge** - `caep_acknowledge_caep_acknowledge__event_id__post` (`POST /caep/acknowledge/{event_id}`, `X-API-Key`, only the DID the event was raised for). Soft-ack; rows are hard-deleted after 90 days.
4. **Re-read and verify** - `get_trust_score_skill_trust_score__did__get` and check `registry_signature` = Ed25519(JCS({did, trust_score, computed_at, valid_until, policy_version})) against the key from step 1. Honour `valid_until`.
5. **Note what this is not.** The provider says so itself: "CAEP" here is a proprietary polling profile inspired by OpenID CAEP, not a SET / RFC 8417 Shared Signals stream. There is no push channel (agent card `pushNotifications: false`).

## Rules an agent must follow

- **Auth.** Send your key in the `X-API-Key` header (free key: `POST /auth/signup` with an email, or `POST /auth/signup-did` by Ed25519 proof of possession - no email, no card). Trust-gated reads accept `X-MolTrust-DID` instead. See `../authentication/moltrust-ch-authentication.yml`.
- **No idempotency key exists anywhere in the 186-operation spec** (`../conventions/moltrust-ch-conventions.yml`, `idempotency.coverage: none`). A timed-out POST that mints something (a registration, a credential, a delegation, a transfer) must be re-read before it is retried, never blind-retried.
- **Credits are spent per call and purchases are final.** `GET /credits/pricing` is the machine-readable cost table (`../plans/moltrust-ch-credits-pricing.json`); most writes cost 1-2 credits, reads like health/resolve/pricing cost 0. The Terms say credit purchases are non-refundable, so a duplicate write is real money.
- **Errors are FastAPI-shaped, not RFC 9457**: `{"detail": ...}` where `detail` is a string (401/400/404) or a `ValidationError[]` array (422). See `../errors/moltrust-ch-problem-types.yml`.
- **Rate limits carry no headers.** Free tier is 60 calls/hour (pricing page) and a 429 arrives with a JSON body and no `Retry-After`. Back off exponentially. See `../rate-limits/moltrust-ch-rate-limits.yml`.
- **A fresh agent's trust score is `null`, not 0**, until it has three endorsements - treat `null` as "unscored", not as a failure.
