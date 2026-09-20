---
generated: '2026-09-19'
method: generated
name: 'EU AI Act: classify a system, issue an Annex V declaration, record an incident'
description: 'Run the compliance API against Regulation (EU) 2024/1689: risk-tier a use case, issue a signed Annex
  V declaration of conformity as a VC, read the report, and record an Article 73 incident with its deadline.'
api: openapi/moltrust-ch-openapi.yml
operations:
- compliance_assess_compliance_assess_post
- compliance_declaration_compliance_declaration_post
- compliance_report_compliance_report__did__get
- compliance_incident_compliance_incident_post
source: Grounded in openapi/moltrust-ch-openapi.yml (captured 2026-09-19 from https://api.moltrust.ch/openapi.json;
  verbatim copy in openapi/_original/). Every operationId verified verbatim by the generating script. Conventions
  per conventions/, errors per errors/, limits per rate-limits/, pricing per plans/.
---

# EU AI Act: classify a system, issue an Annex V declaration, record an incident

Run the compliance API against Regulation (EU) 2024/1689: risk-tier a use case, issue a signed Annex V declaration of conformity as a VC, read the report, and record an Article 73 incident with its deadline.

## Steps

1. **Classify** - `compliance_assess_compliance_assess_post` (`POST /compliance/assess`, `X-API-Key`, body `ComplianceAssessRequest {use_case, intended_purpose, annex_iii_area?, performs_profiling?, ...}`). Returns the tier (prohibited / high / limited / minimal), obligations and the EUR-Lex article each rests on. 2 credits. The provider states this is a protocol-layer determination, not legal compliance.
2. **Declare** - `compliance_declaration_compliance_declaration_post` (`POST /compliance/declaration`, body `ComplianceDeclarationRequest` with all eight Annex V fields; `anchor: true` writes the hash to Base L2). Returns a `MolTrustConformityDeclaration` VC issued by `did:web:api.moltrust.ch`, verifiable against `/.well-known/jwks.json`. 2 credits.
3. **Report** - `compliance_report_compliance_report__did__get` (`GET /compliance/report/{did}`, `X-API-Key` required - an anonymous call returns 422 "Field required" for the header). HTML report of identity, assessment, gaps and declarations.
4. **Record an incident** - `compliance_incident_compliance_incident_post` (`POST /compliance/incident`, body `ComplianceIncidentRequest {did, category, severity?, awareness_date?}`). Computes the Article 73 deadline: 2 days (critical-infrastructure disruption), 10 (death), 15 otherwise. There is no un-record operation; verify inputs before posting.

## Rules an agent must follow

- **Auth.** Send your key in the `X-API-Key` header (free key: `POST /auth/signup` with an email, or `POST /auth/signup-did` by Ed25519 proof of possession - no email, no card). Trust-gated reads accept `X-MolTrust-DID` instead. See `../authentication/moltrust-ch-authentication.yml`.
- **No idempotency key exists anywhere in the 186-operation spec** (`../conventions/moltrust-ch-conventions.yml`, `idempotency.coverage: none`). A timed-out POST that mints something (a registration, a credential, a delegation, a transfer) must be re-read before it is retried, never blind-retried.
- **Credits are spent per call and purchases are final.** `GET /credits/pricing` is the machine-readable cost table (`../plans/moltrust-ch-credits-pricing.json`); most writes cost 1-2 credits, reads like health/resolve/pricing cost 0. The Terms say credit purchases are non-refundable, so a duplicate write is real money.
- **Errors are FastAPI-shaped, not RFC 9457**: `{"detail": ...}` where `detail` is a string (401/400/404) or a `ValidationError[]` array (422). See `../errors/moltrust-ch-problem-types.yml`.
- **Rate limits carry no headers.** Free tier is 60 calls/hour (pricing page) and a 429 arrives with a JSON body and no `Retry-After`. Back off exponentially. See `../rate-limits/moltrust-ch-rate-limits.yml`.
- **A fresh agent's trust score is `null`, not 0**, until it has three endorsements - treat `null` as "unscored", not as a failure.
