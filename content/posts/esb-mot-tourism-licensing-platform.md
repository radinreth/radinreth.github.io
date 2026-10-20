+++
authors = ['Radin']
title = 'ESB-MOT: Building Online Tourism Licensing for Cambodia'
date = '2026-10-20'
summary = "What I learned building the Ministry of Tourism's online licensing platform at TSC: what it changes for businesses and the ministry, how one licence moves through five services, and where the bugs actually came from."
ShowToc = true
TocOpen = false
draft = false
tags = ['esb', 'mot', 'tsc', 'govtech', 'microservices', 'testing']
+++

## Introduction

I'm a software engineer at TSC (Techo Startup Center), and for most of 2026 I've worked on **ESB-MOT**, the Ministry of Tourism's online licensing platform. This post covers what the platform does, why it matters to the people who use it, how it works underneath, and what I would do differently.

I wrote it for three kinds of reader. Engineers joining a government microservices project will find the architecture walk-through and the testing lessons useful. People planning digital public services will find the section on value. Anyone curious about how a restaurant licence gets issued in Cambodia today can read it from the top.

## What ESB is, and what it changes

ESB stands for E-Services for Business. It is a Royal Government of Cambodia programme that gives ministries a shared foundation for digital public services. Most licensing follows the same path. A business applies, an officer reviews, the business pays, and the ministry issues a certificate. TSC maintains an open **generic template** for that path, and each ministry runs its own instance. MOT is the tourism instance, and its first licences cover restaurants, accommodation, sport venues and tour agents.

The value shows up for three groups.

- **Businesses** apply, pay and track their application online instead of visiting an office and carrying paper between desks. Each licence's fee follows a published schedule. For restaurants, the seating-capacity band is the fee line item, so the owner sees the price before submitting.
- **The ministry** gets one backoffice where officers review applications in department-scoped queues. Renewal rules and late penalties are configuration that staff can edit, so a policy change doesn't need a developer.
- **The public** can trust the certificate. The official tourism licence carries a verification QR, and scanning it through verify.gov confirms the licence is genuine. A dedicated service batches payments to the national treasury system (NRMIS), so revenue can reconcile without manual re-entry once that service is deployed.

The template also saves effort across government. Each ministry adapts a working platform instead of commissioning its own from scratch. MOT itself was forked from the agriculture ministry's instance, and the code still carries farm-licence names like `FarmType`. That cost is real, but it is much smaller than starting over.

## How one licence moves through the system

The platform is about twenty independent repos. They contain two Nuxt portals (one for the public, one for the backoffice), Spring Boot services, two Spring Cloud Gateways, shared libraries and infrastructure. Services talk mostly through Avro messages on Kafka.

A new restaurant licence passes through five services.

```
restaurant form (public portal)
  → Nitro proxy → public gateway
  → application-service   (save and submit)
  → Kafka → review-service (officer approves)
  → Kafka → application-service (marks approved)
  → Kafka → profile-service (creates the licence)
  → certificate-service   (renders the PDF with its QR)
```

New applications must be paid before an officer sees them. A platform-wide switch chooses between pay-first-review-later and review-first-pay-later, and changing it changes the whole application lifecycle.

A few design choices keep this manageable.

- **The browser never calls a backend directly.** The portal's own Nitro server proxies every request and turns the httpOnly auth cookie into a Bearer header.
- **Each licence type has a service code and a strategy class.** `GDTI-RT-L` is a new restaurant licence, and adding a licence type means adding strategies rather than growing a `switch`.
- **Kafka messages go out after the database commits.** Business code publishes a Spring domain event, and a transactional listener sends the message after the commit. A rolled-back submission can't start a review for a row that doesn't exist.
- **Message schemas live in one repo.** Services download their Avro schemas from the Schema Registry at build time, so the producer and the consumer read the same contract.

The other operations reuse most of the chain. Renewal runs the editable rule engine and can add a bilingual late-penalty invoice line. Amendment can change the licensee, address and details, but not the capacity band, because the band is the fee. Transfer looks up the recipient's identity, and termination revokes the existing certificate. Transfer and termination skip the payment gate and go straight to review.

## Where the bugs came from

When I looked at the defects that reached our dev environment, very few were inside a single service. Most of them sat on a joint between two correct pieces.

- A portal called an endpoint that the gateway had no route for. Gateway routes are Java classes, so a forgotten route returns 404 before the service ever sees the request.
- A producer and a consumer disagreed about an Avro field. I also lost an afternoon to the tenant prefix. The schema publisher adds its own trailing dot, so passing `esb.` produced subjects named `esb..application_submitted_message-value`.
- A seeder inserted duplicate keys, or a fee row lacked a revenue code that the ministry's workbook listed.
- A guarded endpoint rejected a user who should have had access.

The last case is a good example. Backoffice staff sometimes got a 403 saying they couldn't view applications from other departments, even in their own department. The portal kept the selected department in a module-level singleton and in a localStorage key that wasn't tied to any user. Only the explicit Logout button cleared it. After a failed token refresh, the next user inherited the previous user's department. The backend was right to refuse the request. It had simply received a stale header from the client.

## Bringing tests back

In October 2026 no test ran in CI, and the Jenkins builds skipped tests. Many test files existed on developers' machines but had never been committed. When I brought them under version control on 2026-10-19, the baseline was red in 15 of 16 Maven repos and in both portals, and 23 test classes didn't compile.

We also keep a ledger of deferred defects, and nobody revisits an entry after writing it. I re-checked all 26 open high-severity entries against the current code and databases. Four were already fixed, eleven were confirmed real, and several had a different cause from the one recorded.

The testing strategy we wrote assigns each defect class to the cheapest layer that catches it.

1. Offline unit tests for each service that run without Schema Registry or Docker. CI can't run anything until this layer exists.
2. Message-contract checks that reject an incompatible Avro change.
3. Gateway route coverage, so that every portal call has a route.
4. Persistence tests against real Postgres through Testcontainers.
5. Authorization tests for each role and department.

## What you can take from this

If you work on a similar platform, these are the lessons I'd pass on.

- **Test the joints first.** In a multi-repo system, most defects come from two correct pieces disagreeing. Route coverage and contract tests catch more real bugs than extra unit tests.
- **Make the first test layer runnable anywhere.** If the build needs a running registry or Docker, CI can't run the tests, and a test that never runs drifts until it no longer compiles.
- **Re-verify a bug list before you plan from it.** An entry records what someone believed on the day they wrote it.
- **Tie client-side context to the user and the session.** Any state the client sends as a header must be cleared on every way a session ends, not only the Logout button.
- **Trace one flow end to end before you change anything.** I wrote a hop-by-hop trace for a teammate coming from Rails, and it was the most useful document I produced all year.
- **Put policy in configuration.** Fees, eligibility rules and penalties change with ministry decisions. When staff can edit them, the platform keeps up with policy without a release.
