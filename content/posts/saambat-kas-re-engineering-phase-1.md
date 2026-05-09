+++
authors = ['Radin']
title = 'Re-engineering the KAS Platform — Phase 1'
date = '2026-05-09'
summary = "Notes from four months re-architecting SAAMBAT/KAS at TSC: replacing the gateway, dropping internal E2EE, consolidating IAM, standardizing observability, and putting Kafka in the middle."
ShowToc = true
TocOpen = false
draft = false
tags = ['saambat', 'kas', 'tsc', 're-architecture']
+++

## Background

SAAMBAT is a Cambodia-government program supporting agricultural value chains, and KAS is the digital platform behind it. I've been working on it at **TSC** (Techo Startup Center). The platform grew organically over a few years — the kind of growth that ships features but accumulates the usual debts: a Node.js gateway no one fully owned, encryption layered between services that never left the cluster, two separate role tables, and a logging story that depended on whichever engineer set up the service first.

We kicked off a re-engineering program on **2026-01-01** with a clear scope for Phase 1: stop bleeding before adding any new industries. This post is a personal write-up of what shipped between January and the end of April.

## The five things that mattered

### 1. The gateway is now Spring Cloud Gateway

The old `kas-core` was a Node.js proxy doing JWT validation, partner E2EE, and routing. We replaced it with **Spring Cloud Gateway** on port 3000, with JWT validation moved to the edge using HS256 with a secret shared with `kas-iam`. Downstream services now verify locally; the `/authenticate-jwt` round-trip is gone.

Path rewriting is split cleanly: `/api/*` for internal traffic, `/integrations/*` for partners. CORS is centralized — localhost only allowed in non-prod profiles, and the Helm chart defaults to the `prod` Spring profile as a fail-safe.

It's running in dev. Staging and production are waiting on DevOps to provision the new cluster.

### 2. We removed internal end-to-end encryption

This one took some convincing. The previous design had **secp256k1 + AES-256-CBC** between every internal service. For traffic that never leaves the cluster, the operational complexity was real and the threat model was thin.

So we ripped it out internally and kept **partner-facing E2EE**, which we moved out of `kas-core` and into `kas-service-registry` — that's where partner key material already lives. The `@kas/e2ee-utils` package stays for partner integrations.

Net effect: simpler internal topology, partner trust boundary unchanged.

### 3. IAM and RBAC, finally consolidated

The single biggest defect I personally worked on was a duplicate-roles bug. **Migration 001** had created agriculture-scoped roles without removing pre-existing duplicates. The result: `createUser` validation broke in surprising ways. **Migration 003** deduplicates the roles, and `createUser` now runs Set-based dedup as defense in depth.

Auditing the permission model produced a useful surprise: per-service permission tables were dead data. The frontends drive authorization off **roles**, not individual permissions. Only `USER.*` permissions were actually enforced anywhere. So those tables go on the housekeeping pile.

Two smaller fixes in the same pass — the `kas-iam` Dockerfile now ships the `public/` directory (email templates), and `EmailController` no longer crashes when a template file is missing.

### 4. Observability — one pattern, every service

We picked a single transformation pattern and applied it everywhere:

1. **Winston** structured (JSON) logs
2. **Correlation IDs** end-to-end (HTTP `X-Correlation-ID`, plus the Socket.io equivalent)
3. **Sentry** for errors
4. **Slack alerting** for 5xx with full context
5. **ELK** for centralized log aggregation

All eight backend services are on the pattern. Some are further along on OpenAPI documentation and Vitest coverage than others — `kas-iam` (43 endpoints), `kas-service-registry` (13), and `kas-weather-api` (107) are documented; others still need that pass.

### 5. Kafka as the event backbone

- **Kafka 3.6**, three brokers, KRaft mode (no ZooKeeper).
- **26 topics** across portal, chat, notification, weather, resource, IAM, registry, and DLQ categories.
- Naming standardized as `kas.<service>.<entity>.<event-type>`.
- A `kafka-init` service auto-creates topics on startup; `validate-topics.sh` for health checks.
- **Schema Registry 7.5.0** deployed (though contracts are not yet enforced — a known gap).
- Four DLQ topics in place; replay tooling is still manual, which is on the Phase-2 list.

## Local development that doesn't fight you

A small thing that mattered a lot: the unified Docker Compose stack in `kas-infrastructure/`. One command brings up MongoDB replica set, Redis, Kafka cluster, Schema Registry, ELK, and all eight backend services on documented ports. `start.sh` has `--build` and `--no-cache` flags; `reset.sh` got a `--deep`.

We also renamed `docker-compose.override.yml` → `docker-compose.staging.yml`. The old name was an automatic include — local dev was occasionally pointing at the staging MongoDB by accident. Now staging is opt-in via `--staging`.

## Notable bugs along the way

- **BUG-3 (form-data on `kas-resource-api`)**: farmer / family-member submissions were broken. Root cause was `formidable 2.1.5` auto-parsing bracket notation while our manual reconstruction code did the same job — they fought.
- **kas-weather-api**: form handlers now have a fallback path, and error responses no longer leak internals.

## What's left for Phase 1

These are operational, not architectural:

- **DevOps** to provision the new cluster so the gateway can promote dev → staging → production. Once production cuts over, the legacy `/authenticate-jwt` endpoint can be retired.
- **Housekeeping**: sunset `kas-core` images and CI jobs, drop dead per-service permission tables, remove unused backend routes catalogued in `docs/re-architecture/unused-backend-routes.md`, scrub `kas-core` references from service `CLAUDE.md` files.

## What we'd need for nation-wide scale

Phase 1 got us internally consistent. Going from there to production-grade for a nation-wide rollout is a different list — it's the one I think about most. The headline items:

- **Resilience**: multi-AZ MongoDB and Kafka, circuit breakers (Resilience4j on the gateway), automated DLQ replay, DR runbooks with documented RTO/RPO.
- **Security**: Vault-backed secrets, mTLS between gateway and downstream, rate limiting per principal/route, WAF, PII masking extended beyond `kas-iam`, a real penetration test, and a Cambodia data-protection compliance review.
- **Observability beyond logs**: OpenTelemetry traces, Prometheus + Grafana golden signals, formal SLOs (we have SLA targets but not SLOs), synthetic monitoring from regional vantage points, and an actual on-call rotation rather than Slack-only alerting.
- **Performance**: load testing with k6 or Gatling, a capacity plan sized to provinces × users × forms/day, CDN, MongoDB read replicas, Redis cluster mode, a connection-pooling sweep.
- **Data governance**: enforce Schema Registry contracts on `kas.*` topics, adopt one migration tool with idempotency tests, define retention policies, and inventory PII.
- **DX & CI/CD**: split CI matrices, contract tests at the gateway boundary, ephemeral pre-merge environments, branch protection and conventional commits, an internal developer portal.
- **Frontend & mobile**: offline-first hardening with proven sync conflict handling, low-bandwidth optimizations, Khmer language QA, and WCAG AA for backoffice tools used by government staff.
- **Operational readiness**: a real tenant model (province → district → commune as first-class citizens), feature flags for staged regional rollout, bulk import/export tooling for legacy records, training materials, and helpdesk integration.

## Phase 2, if it happens

There's a proposal on the table to generalize KAS beyond agriculture into a **multi-industry data-entry backbone**. We've done the preparation work — catalogued every place the codebase assumes "agriculture," drafted the generic submission envelope:

```json
{
  "tenantId": "...",
  "industry": "...",
  "formId": "...",
  "schemaVersion": 1,
  "payload": { },
  "attachments": [ ],
  "geo": { },
  "submittedBy": "..."
}
```

…and confirmed Schema Registry can host industry-specific payload contracts. The architectural precondition is **extracting the form engine** out of `kas-resource-api` into a runtime form-definition service, with industry packs (form definitions + RBAC seed + reference data + optional industry service) as a first-class concept.

But that's a leadership decision. Engineering's recommendation is a small, scoped extraction of the form engine first — about a quarter, two or three engineers — before committing to a second industry.

## Reflection

The most useful thing I learned in Phase 1 isn't on the report. It's that **picking one transformation pattern and applying it everywhere** beats tailoring per-service. The observability rollout went fast precisely because we stopped re-litigating the design at each service. Same for the Docker Compose unification — once the canonical stack existed, every service became "fit into this" rather than "design your own."

The risk for Phase 2, if leadership greenlights it, isn't technical. It's the temptation to keep adding capabilities in parallel with the multi-industry generalization. That's the way I get a hybrid platform that does neither well. So Phase 2 should ship behind a feature flag, with one new industry as the proof, before anything else.
