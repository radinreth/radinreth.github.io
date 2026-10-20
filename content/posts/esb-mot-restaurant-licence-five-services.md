+++
authors = ['Radin']
title = 'ESB-MOT: One Restaurant Licence, Five Services'
date = '2026-10-20'
summary = "Tracing a single restaurant licence application through the Ministry of Tourism's ESB platform, from a Nuxt form to a signed PDF, across five Spring Boot services and two Kafka hops."
ShowToc = true
TocOpen = false
draft = false
tags = ['esb', 'mot', 'tsc', 'spring-boot', 'kafka', 'microservices']
+++

## Background

ESB stands for E-Services for Business. It is a Royal Government of Cambodia programme that gives ministries a shared foundation for digital public services. The flow is the same for most of them. A business submits an application, an officer reviews it, the business pays, and the ministry issues a certificate. At TSC (Techo Startup Center) we maintain an open generic template for this, and each ministry gets its own instance.

I work on the **MOT** instance, for the Ministry of Tourism. Its first licences cover restaurants, accommodation, sport venues and tour agents. MOT was forked from the agriculture instance, so the code still carries farm-licence names like `FarmType` and `FarmSize`. I learned early to read those as inherited scaffolding rather than intent.

The workspace is about twenty independent GitHub repos checked out side by side. Each service, portal, shared library and infrastructure repo has its own trunk, version and release cycle. A change that touches three services means three coordinated commits, and often a library version bump and a Helm chart bump on top.

This post follows one restaurant registration from the form to the PDF. I wrote the original trace for a teammate coming from Rails, and it turned out to be the most useful document I produced this year.

## The shape of it

A restaurant owner fills in a form and picks a seating-capacity band, which is also the fee line item. They submit it. New applications must be paid before an officer ever sees them. That rule comes from a global `PAYMENT_MODE` switch, which selects pay-first-review-later or review-first-pay-later for the whole platform. Changing it changes the application lifecycle, not just a label.

An officer reviews and approves the application in **review-service**. Approval fires a Kafka message that **profile-service** consumes to create the licence record. Profile-service then asks **certificate-service** to render the PDF.

In Rails this would be one controller and an `after_commit` callback. Here it is five Spring Boot processes, three databases and two Kafka hops, and no single log file shows the whole story.

```
restaurant form (Nuxt public portal)
  → repositories/application/request-application.repo.ts
  → server/api/[...].ts            (Nitro proxy)
  → public-gateway-service         (Spring Cloud Gateway)
  → application-service            (save, then submit)
  → Kafka: application_submitted_message
  → review-service                 (officer approves)
  → Kafka: review_request_created_message
  → application-service            (marks approved)
  → Kafka: application_approved_message
  → profile-service                (creates the licence)
  → certificate-service            (renders the PDF)
```

## Hop by hop

### The portal never calls a backend directly

Every endpoint lives in a `repositories/*.ts` file, and `plugins/api.ts` registers each repository on one shared fetcher. That fetcher attaches auth headers and handles the response envelope in one place. A repository file that exists but is not registered is unreachable. You get `undefined is not a function` instead of a 404, and this is the most common reason a new endpoint "doesn't work."

The request then goes to the portal's own Nitro server. A catch-all route at `server/api/[...].ts` rewrites `/api/**` to the gateway URL and turns the httpOnly `access_token` cookie into a Bearer header. The browser never sees a backend address.

### Gateway routes are Java classes

Spring Cloud Gateway can be configured in YAML, but ours isn't. Each downstream service has a `*RouteConfiguration` class with path predicates, a token-requirement filter and an authorization filter. If you add an endpoint and forget the route class, the gateway returns 404 and the service never sees the request. This failure mode comes up often enough that I now check the route class before I check anything else.

### application-service dispatches on a service code

Every licence type has a service code, and `GDTI-RT-L` is a new restaurant licence. `ApplicationStrategyFactory` builds a map from service code to strategy bean, so a restaurant application ends up in `TourismRestaurantLicenseStrategy`. Renewal, amendment, transfer and termination each get their own strategy. Adding a licence type means adding strategies rather than editing a growing `switch`.

### Kafka messages go out after the commit

Business code never calls a Kafka producer directly. It publishes a Spring domain event such as `ApplicationSubmittedDomainEvent`, and a `@TransactionalEventListener` sends the Kafka message after the database transaction commits. A rolled-back submission therefore can't leak a message that tells review-service to start work on a row that doesn't exist.

### Schemas live in one repo

The Avro `.avsc` files live in `infrastructure/schema-registry`, not in the services. Each service lists the subjects it needs in its `pom.xml`, and the build downloads them from the Schema Registry and generates Java classes. A service won't compile without a reachable registry. It also needs the `KAFKA_TENANT_ID_PREFIX` variable exported, even when the prefix is empty.

I lost an afternoon to the tenant prefix. The publisher adds the trailing dot itself, so passing `esb.` produces subjects named `esb..application_submitted_message-value`. Every service build then 404s on its subject, and the error message points at the registry rather than at the double dot.

### Same class name, different repo

Profile-service has its own `TourismRestaurantLicenseStrategy`. It has the same name as the one in application-service, lives in a different repo, and does a different job. The application-service strategy validates and stores the application. The profile-service one turns an approved application into a licence. Searching the workspace for the class name returns both, and new readers routinely open the wrong one.

### The certificate

Certificate-service renders the licence through jsreport with Khmer fonts and templates. The official MOT tourism licence also carries a verification QR, which anyone can scan to confirm the licence is genuine through verify.gov. Most of the code for that already existed when I picked it up. The restaurant certificate row was seeded with verification switched off, so the QR block never rendered. Turning it on was mostly a matter of wiring up external endpoints and credentials and removing configuration that the code never read.

## What differs for the other operations

Registration is the longest path, so I traced it first. The other four operations reuse most of the chain.

- **Renewal** runs a rule engine before it bills. The engine is versioned and editable in the backoffice, and it can add a late-renewal penalty that payment-service prints as a bilingual invoice line.
- **Amendment** can change the licensee, address, restaurant details and inspection. It can't change the capacity band, because the band is the fee. A licensee who needs a different band has to terminate and register again.
- **Transfer** looks up the recipient's identity before profile-service moves the licence.
- **Termination** doesn't issue a new certificate. It revokes the existing one.

Transfer and termination skip the payment gate entirely and go straight to officer review. Eligibility is enforced by the same rule engine. An expired licence can't be amended or transferred, a terminated one can't be renewed, and a licence can't be terminated twice. Restaurant supplies these rules as configuration rows rather than code.

## What I'd tell someone starting on it

Read one flow end to end before you change anything. The services are small and conventional on their own. The difficulty is at the joints between them, which are gateway routes, Avro subjects, Kafka topics and the department header that the backoffice portal sends on every request. Most of the bugs I fixed this year sat on one of those joints. The next post covers how we started testing them.
