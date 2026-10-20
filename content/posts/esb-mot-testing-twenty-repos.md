+++
authors = ['Radin']
title = 'ESB-MOT: Testing a Platform Spread Across Twenty Repos'
date = '2026-10-20'
summary = "MOT had no tests running in CI, and the defects that reached dev clustered at the boundaries between services. Notes on re-verifying a stale bug ledger, a stale department header, and a testing strategy built around where defects actually land."
ShowToc = true
TocOpen = false
draft = false
tags = ['esb', 'mot', 'tsc', 'testing', 'kafka', 'nuxt']
+++

## Where we started

The MOT platform ships through about twenty independent repos that talk to each other over two gateways and Avro messages on Kafka. In October 2026, no test ran in CI, and the Jenkins builds skipped tests. Many test files existed on developers' machines but had never been committed, because a local exclude kept `src/test` out of git.

On 2026-10-19 I brought the existing tests under version control. The baseline that day was red in 15 of 16 Maven repos and in both portals, and 23 test classes didn't compile at all against the current code.

That number sounds bad, but it was useful. It told us the tests had drifted from the code for months without anyone noticing, which is exactly what happens when nothing runs them.

## Defects live at the boundaries

Before writing a strategy, I looked at what actually broke on our dev environment. The defects clustered in four places, and per-class unit tests catch none of them.

- **Unrouted gateway paths.** A portal calls an endpoint the gateway has no route for, and the request 404s before it reaches the service.
- **Schema mismatches.** A producer and consumer disagree about an Avro field or a union default.
- **Department-scope 403s.** A guarded endpoint rejects a user who should have access, or accepts one who shouldn't.
- **Seed-data drift.** A seeder inserts duplicate keys, or a fee row is missing a revenue code that the ministry's workbook says should exist.

The developers maintaining MOT, including me, were finding these by hand during smoke passes, one at a time.

## Re-verifying the bug ledger

We keep a ledger of deferred defects. Someone writes an entry when they find a problem, and nobody revisits it afterwards. By September there were 26 open entries marked high severity. I checked every one against the current code and the live local databases instead of trusting the entry text.

| Outcome | Count |
|---|---|
| Stale, already fixed by later work | 4 |
| Fixed and shipped the same day | 2 |
| Confirmed real and reachable | 11 |
| Real, fix written but parked | 1 |
| Real but not reachable today | 3 |
| Real but deliberate design | 1 |
| Not verified | 3 |
| External, not an engineering task | 1 |

Four entries described problems that no longer existed. Several of the confirmed ones had a different cause from the one recorded. One "small data task" turned out to need an ownership ruling from the ministry and a code change, because the penalty revenue code was derived as `serviceCode + "-PENALTY"`. That formula can't tell two seating-capacity bands apart, and the ministry bills them under different treasury codes.

My conclusion is that a defect ledger rots like any other documentation. An entry records what someone believed on the day they wrote it. It needs re-verification before anyone plans work from it.

## The stale department header

One bug shows the boundary problem clearly. Backoffice staff would occasionally open the applications list and get a 403 with the message key `unable.to.view.applications.from.other.departments`, even though they belonged to the department they were viewing.

The cause was in the portal, not the backend. A user's permissions are scoped to a department, and the portal sends an `X-Department-ID` header on every request. The selected department lived in a module-scope `ref`, which is a singleton for the whole app rather than per login. It was also persisted under one fixed localStorage key that wasn't tied to any user.

A watcher auto-selected the first department for a newly loaded user, but only when nothing was selected yet. If the previous user's department was still in localStorage, the watcher never fired. The new user's departments loaded correctly but were never applied, and every request went out with someone else's department ID.

The stored value was only cleared on the explicit Logout menu action. A failed token refresh, a redirect from the auth middleware, or a closed browser all skipped it.

The backend did its job by refusing a department the user didn't belong to. The bug was a correct server-side check receiving a stale client-side header. Reproducing it was harder than explaining it. The obvious reproduction is two real CamDigiKey logins in one browser, but we had no local CamDigiKey mock and only one working identity. The auth cookies are httpOnly and secure, so no tool could inject a second user's session without adding a bypass route to the portal, and we ruled that out. The reproduction became a test against the composable itself. It puts a stale department in localStorage, gives the composable a different user's department list, and asserts that the header value changes. It needs no login, cookie or browser, and the same test serves as the regression check for the fix.

## The strategy

The testing strategy assigns each recurring defect class to the cheapest test layer that catches it.

1. **Offline fast tests per service.** Unit and slice tests run with Schema Registry and Docker stopped. Anything that needs a database or Kafka is tagged and runs in a separate tier.
2. **Message-contract checks.** The schema publisher refuses an incompatible change to a field or union default. A producer whose serialized message no longer matches its registered subject fails a round-trip test, with or without the tenant prefix.
3. **Gateway route coverage.** Removing a path from a route class, or adding a portal call with no matching route, fails a check.
4. **Real-database persistence tests.** Repository, Specification and native SQL queries run against real Postgres through Testcontainers. A mocked repository would pass a broken column name. A real database won't.
5. **Authorization and department-scope tests.** Guarded endpoints are verified for each relevant role and department.

The order matters. The first layer has to exist before anything goes into CI, because a suite that needs a running registry to compile can't run on a Jenkins agent that has none. One service, nrmis-service, couldn't even build in CI. Its `pom.xml` pulled Avro schemas from a sibling checkout that existed only on one developer's laptop, so it had no Jenkins job and ArgoCD deployed nothing for it.

## What I took from it

Unit test coverage was never the main gap. Most of the classes we tested were simple and correct. The defects that hurt us came from two correct pieces that disagreed, such as a portal and a gateway, a producer and a consumer, or a seeder and a ministry spreadsheet. A strategy for a platform like this has to start from where the defects actually landed, and on MOT that was the joints between repos.
