# Privacy and compliance

> **Status: draft.** Observa is in early development and nothing described here is live yet. This document describes the planned design. It is not legal advice and has not yet had an independent legal review (see [Open items](#open-items)).

This document is for tenant projects, their data protection contacts, and SCF reviewers. It explains what data Observa collects, what it never collects, how long data is kept, and how Observa plans to meet the EU General Data Protection Regulation (GDPR). The technical design behind it is in [architecture.md](architecture.md#privacy-and-gdpr-approach).

## Summary

| Observa does | Observa does not |
|--------------|------------------|
| Count page views, visitors and API requests | Set cookies or use local storage in visitors' browsers |
| Store country-level location | Store raw IP addresses, regions or cities |
| Count unique visitors with a hash that changes every day | Track visitors across days or across websites |
| Store normalised API routes, status codes and latency | Store request bodies, headers, client IPs, traces or logs |
| Host all data in the EU (proposed: Hetzner, Germany) | Sell, share or use data for advertising |
| Record wallet connect and sign events if the tenant opts in | Record wallet addresses, transactions or signatures |

## Data collected: web analytics

| Data | Source | Stored as |
|------|--------|-----------|
| Website identifier | Tracker configuration | As sent |
| Timestamp | Server clock | As recorded |
| Page path | Page URL | Path only. Query strings are removed by default, except an allow-list of campaign parameters such as `utm_source` (proposed). |
| Page title | Document title | Proposed off by default, because titles can contain personal data. See open items. |
| Referrer | Browser referrer | Domain and path only, query string removed (proposed) |
| Country | Truncated IP address and a local GeoIP database | Two-letter country code. Region and city are not stored. |
| Browser, operating system, device type | User agent and screen size | Category names (for example "Firefox", "Linux", "desktop"). The full user agent is not stored. |
| Screen size and browser language | Read by the tracker script | Whether to keep, coarsen or drop these is an open item |
| Daily visitor hash | Website, truncated IP, user agent and a daily secret salt | One-way hash. The salt is deleted after the day ends. |
| Custom events (optional) | Tenant's own code | Event name and properties chosen by the tenant |
| Stellar Wallets Kit events (optional) | Tracker | Event name and wallet type only |

The full IP address exists only in memory while a request is processed. Observa truncates it first (IPv4 last octet removed, IPv6 cut to /48). The truncated address is used for the country lookup and the daily hash, and then discarded. The short-lived abuse check described in [architecture.md](architecture.md#bot-and-inflation-mitigation) keeps only a keyed hash in memory, and that hash expires within minutes.

## Data collected: API analytics

Observa keeps only:

- Tenant (set from the ingest key)
- Service name
- Normalised route (for example `/accounts/{address}`)
- HTTP method
- Status code
- Request counts and latency histogram buckets, per minute

Everything else in the OpenTelemetry data is dropped before storage: spans, traces, logs, request and response bodies, headers, client IP addresses, user identifiers, and all other attributes. Path normalisation also replaces Stellar addresses, numeric IDs, UUIDs and hashes in paths with placeholders.

## Data collected: tenant and operator accounts

To run the service, Observa stores some data about the people and projects that use the dashboards:

| Data | Purpose |
|------|---------|
| Stellar public keys used to sign in | Authentication and access control |
| Project name, domain, and Stellar account and contract IDs | Enrolment and verification. These are already public on the domain and on-chain. |
| Audit log entries (which key did what, and when) | Accountability for operator and administrative actions |
| Dashboard session | A strictly necessary sign-in session for dashboard users only. Website visitors never get a session cookie. |

No password or email address is required. Whether to collect an optional contact email for security and service notices is an open item.

## Roles under GDPR

The proposed roles, subject to legal review:

- For **visitor and API analytics data**, each tenant is the **controller** (it decides to measure its own website or API), and the Observa operator is a **processor** acting on the tenant's instructions.
- For **tenant and operator account data**, the Observa operator is the **controller**.

A data processing agreement (DPA) template for tenants is planned.

## Lawful basis and consent

**GDPR.** Observa is designed so that the visitor data it stores is not personal data: IP addresses are truncated and discarded, and the visitor hash can't be linked across days or reversed once the daily salt is deleted. Where a lawful basis is still needed, the tenant as controller chooses it. For privacy-preserving aggregate analytics this is usually legitimate interests, but that is the tenant's assessment to make.

**ePrivacy (cookie rules).** Observa sets no cookies and writes nothing to the visitor's device. However, the tracker script does read some information from the browser, such as screen size and language. European Data Protection Board Guidelines 2/2023 take a broad view of what counts as "accessing" information on a device. Whether Observa's tracker needs consent under national ePrivacy rules is therefore a question for the planned legal review. Reducing what the tracker reads is one option under consideration. Until that review is done, Observa does not claim that no consent banner is needed in every EU country.

## Data residency and sub-processors

All analytics data is planned to be stored and processed in the EU.

| Provider | Purpose | Location | Receives analytics data? |
|----------|---------|----------|--------------------------|
| Hetzner (proposed) | Hosting, database, backups | Germany | Yes |
| External uptime check provider (to be chosen) | Checks that public endpoints respond | To be decided | No |
| GitHub | Source code, issues, releases | United States | No |

GeoIP lookups use a database file stored on Observa's own servers. No external lookup service is called. A full sub-processor list will be published before launch, and tenants will be told about changes.

## Retention

| Data | Retention |
|------|-----------|
| Raw API metrics (per minute) | Short troubleshooting window (proposed 30 days) |
| Hourly and daily rollups | At least 24 months |
| Web events | To be decided (see [architecture.md](architecture.md#retention)) |
| Daily visitor-hash salt | Deleted when the day ends |
| Full IP addresses | Never stored |
| Backups | Rotated on a fixed schedule (to be decided) |
| Tenant data after account deletion or removal | Deleted within a defined period (to be decided), including from backups as they rotate out |

## Data subject rights

Because visitor data is designed not to identify anyone, Observa usually can't link stored data to a specific person. GDPR Article 11 covers this case: a controller doesn't have to collect extra data just to identify someone in order to answer a rights request. Observa will not add identifiers to make this possible.

For cases where personal data does end up stored, for example a tenant puts personal data into custom event properties despite the documentation, Observa plans a **tenant-triggered erasure workflow**. It will let a tenant delete data by website, service, time range or custom event property.

Tenants can delete their account, and all analytics data linked to it is deleted with it.

## Tenant responsibilities

- Do not send personal data in custom event names or properties, URLs, page titles or route names.
- Set `http.route` in API middleware where possible, so that raw paths with identifiers are not sent.
- Mention Observa analytics in your own privacy notice.
- Make your own assessment of lawful basis and ePrivacy requirements in your jurisdictions.

## Personal data breaches

If a breach affects tenant data, the Observa operator will notify affected tenants without undue delay, as GDPR Article 33(2) requires of processors, so they can meet their own reporting duties. The incident process is described in [security.md](security.md#incident-response) and in `ops/runbooks/`.

## Planned compliance documents

- Data processing agreement template for tenants
- Records of processing activities (GDPR Article 30)
- An assessment of whether a data protection impact assessment (DPIA) is needed, and the DPIA itself if so
- Published sub-processor list
- Privacy notice for Observa's own website and dashboards

## Open items

| Item | Decision needed |
|------|-----------------|
| Legal review | Independent review of this document, the GDPR roles and the ePrivacy position |
| Tracker data | Whether to keep, coarsen or drop screen size and browser language |
| Page titles | Whether page title collection is off by default or removed |
| Query strings | Final allow-list of campaign parameters |
| Contact email | Whether to collect an optional tenant contact email |
| Deletion periods | Time limits for deleting data after account removal, and backup rotation |
| Uptime provider | Choice and location of the external uptime check provider |
| DPA and DPIA | Drafting the DPA template; assessing whether a DPIA is needed |
