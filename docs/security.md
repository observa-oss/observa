# Security

> **Status: draft.** Observa is in early development and nothing described here is live yet. This document describes the planned security design. To report a vulnerability, see [SECURITY.md](../SECURITY.md).

This document is for SCF reviewers, tenants and security researchers. It explains what Observa protects, the main threats, and the planned controls.

## What Observa protects

1. **Integrity of the numbers.** Observa's purpose is trusted usage data, so preventing fake or inflated data matters as much as keeping data confidential.
2. **Separation between tenants.** No tenant can read or write another tenant's data.
3. **The tracker script.** It runs on every tenant website, so tampering with it would affect all of them.
4. **Credentials and keys.** These include ingest keys, consumer API keys, operator access, release signing and the visitor-hash secret.
5. **Availability.** Observa is not in any tenant's critical path, but SCF relies on its data.

## Threats and planned controls

| Threat | Planned controls |
|--------|------------------|
| A tenant reads another tenant's data | PostgreSQL row-level security on `tenant_id`; the application database role can't bypass RLS; dedicated isolation tests in `tests/` |
| A tenant writes data as another tenant | The tenant is set from the authenticated ingest key and never from submitted data |
| Fake or inflated web traffic | Bot user-agent filtering, browser-execution proof, fingerprints-per-IP checks (see [architecture.md](architecture.md#bot-and-inflation-mitigation)) |
| Tampered tracker script | Version-pinned URLs, Subresource Integrity hashes, signed releases, no automatic updates |
| Compromised dependency or build | Pinned dependencies and lockfiles, reviewed updates, SBOMs, signed artifacts, reproducible builds where practical |
| Stolen operator access | Wallet-based sign-in (hardware wallets recommended for operators), short sessions, audit log of operator actions |
| Leaked ingest or consumer API key | Keys stored only as hashes and shown once; scoped; revocable at any time; rate limited |
| Flooding of ingestion endpoints | Per-key and per-IP rate limits, request size limits, load balancer limits; tenant applications are unaffected because sending is asynchronous |
| Compromise of hosting or source-code accounts | Multi-factor authentication on GitHub and hosting accounts, least-privilege access, branch protection with required reviews |
| Data loss | Database standby replica, encrypted backups in the EU, regular restore tests |

## Authentication and authorisation

- **Dashboard users** sign in with a Stellar wallet using SEP-10 (classic accounts) or SEP-45 (contract accounts). The resulting session is short-lived and held in a secure, HTTP-only, same-site cookie. This cookie is for signed-in dashboard users only. Website visitors never receive a cookie.
- **Roles:** tenant member, SCF operator, and machine consumer. How SCF operators are designated is an open item. Every change to the operator list is audited.
- **Ingest keys** (for API telemetry) belong to one tenant and are sent with each OTLP request.
- **Consumer API keys** are created and revoked by SCF operators, give read-only access to approved views, and never give direct database access.

## Secrets and keys

- Ingest keys and consumer API keys are high-entropy random values, stored only as hashes and shown once when created.
- Infrastructure secrets are kept out of the repository and stored encrypted. The tool to use is an open item.
- The daily visitor-hash salt is generated automatically and destroyed when the day ends.
- Release signing is planned with Sigstore/cosign keyless signing tied to the project's CI identity, which avoids a long-lived private signing key.

## Network and transport

- TLS on all public endpoints, including OTLP over gRPC and HTTP
- The database is reachable only on a private network, never from the internet
- Host firewalls allow only required ports
- Dashboards use standard browser security headers, such as a Content Security Policy and HSTS

## Application and development security

- Input validation and request size limits on all ingestion endpoints
- All changes go through pull requests with review
- Automated dependency, secret and static-analysis scanning in CI (tools to be chosen)
- Upgrades of the upstream projects (Umami, OpenTelemetry Collector) are deliberate, reviewed changes to a pinned version

## Logging and audit

- Operational logs never contain raw IP addresses, keys or request bodies
- An audit log records operator actions: creating and revoking keys, removing tenants, and changing roles
- Monitoring and alerting are described in [architecture.md](architecture.md#observability)

## Incident response

The planned process:

1. **Detect.** Alerts, external uptime checks or a vulnerability report.
2. **Contain.** For example, revoke keys, block traffic or take a component offline.
3. **Assess.** Work out what data and which tenants are affected.
4. **Notify.** Tell affected tenants without undue delay (see [privacy.md](privacy.md#personal-data-breaches)) and publish an advisory where appropriate.
5. **Recover and review.** Fix the cause, restore service, and publish a short post-incident review.

Step-by-step runbooks will live in `ops/runbooks/`.

## Independent review

No external security audit or penetration test has been done yet. Whether one is commissioned before or during the hosting period is an open item.

## Open items

| Item | Decision needed |
|------|-----------------|
| Operator designation | How SCF operators are appointed and removed |
| Secrets management | Tool for encrypted infrastructure secrets |
| CI scanning | Choice of dependency, secret and static-analysis scanners |
| Rate limits | Default limits per ingest key, per consumer key and per IP |
| External audit | Whether and when to commission a security audit or penetration test |
| Security contact | Reporting address and response times (see [SECURITY.md](../SECURITY.md)) |
