+++
authors = ['Radin']
title = 'CamWallet: What Two Retrospectives Taught Me About Building With an Agent Loop'
date = '2026-10-20'
summary = "CamWallet's v2 epics were built with an unattended AI agent loop. Two retrospectives showed the same pattern each time: the loop built too much, a human review cut it back, and the history recorded both."
ShowToc = true
TocOpen = false
draft = false
tags = ['camwallet', 'tsc', 'ai-assisted-development', 'retrospective', 'process']
+++

## The setup

For CamWallet v2 I drove implementation through the BMad Method and a CLI called bmad-loop. BMad is a set of AI agent skills for planning, writing stories, implementing and reviewing. bmad-loop runs the implementation phase unattended. It picks the next story from a sprint file, writes a spec, implements it, runs a verify gate and moves on. Our verify gate was a backend compile plus a frontend typecheck.

I ran retrospectives after Epic 1 (merchant management) and Epic 3 (scheme participation). Both epics shipped and both were accepted. Both retrospectives also found the same problem from different directions, so I'm writing them up together.

## Epic 1: built, then removed

Epic 1 had seven stories. The unattended run implemented all of them, along with seven capabilities that a pilot didn't need. These included member management, a module API, an idempotency key, a member National ID field, server-side search and paging, merchant status and reference uniqueness.

Human review took them back out. The squashed feature commit was about 2,500 lines. After the slim-down, the net change against `dev` was about 550 lines across 13 files. Tests dominated the churn. One service test file gained 1,482 lines and lost 1,322.

Each removed capability became a tracked deferral, so the work wasn't wasted as a record of intent. It was wasted as build time and review time.

The retrospective produced four process lessons.

1. **Slim the scope before building, not after.** Before the next run, I would mark each story's pilot-minimal scope and its deferrals in the epic context file, and apply a minimal-test rule of one service test per rule and one controller test per route.
2. **The loop's independent review never ran.** It was set to trigger only when recommended, and no pass recommended it. Every defect in Epic 1 was found by a human. I had to decide whether to make the loop review always or keep human review as the gate on purpose.
3. **Don't rewrite history during review.** I squashed and force-pushed while the PR was open, and the commit references in our docs moved six times. During review I should add commits and squash once, at merge.
4. **Clean up after the run.** Archive the paused run and mark the epic done in the sprint file.

## Epic 3: same pattern, smaller

Epic 3 was smaller, with three stories and a net change of 4 files and about 224 added lines. I had applied the Epic 1 lessons, so I expected a clean run. It wasn't clean.

About 250 lines landed on `dev` and never survived. One story added a custom JSON deserializer and the next story removed it. A 104-line controller test grew to 151 lines and was then deleted by a refactor commit. The retrospective traced this to five causes.

- **The epic started on a stale base.** `dev` had already gained a component that did the deserializer's job, but the plan still said to build one, and the agent followed the plan. It built what the plan described without checking what the code already did.
- **"Minimal tests" failed in both directions.** The first pass wrote about 480 test lines despite the rule. The trim then cut too deep and dropped the only assertions that checked membership and roles. The rule said how many tests to write but not which ones mattered.
- **Memory pressure killed the run twice.** Each kill needed a manual rollback. Accepting the committed work as a new baseline then mismatched the spec's recorded baseline, so the loop deferred a story that was already finished.
- **A proposal encoded a wrong framework assumption.** It planned to catch a wrapped `DbActionExecutionException`, but Spring Data JDBC throws `DuplicateKeyException` directly. The test suite caught it.
- **Four course-correction proposals in two days** reshaped one small epic. The merged commits didn't line up with story boundaries.

## The pattern

Both retrospectives show an agent loop that is good at producing code that matches a plan and poor at noticing when the plan is out of date. It doesn't push back on scope, so it builds everything the plan mentions. It doesn't check the current branch, so it rebuilds things that already exist. It treats a test-count rule as a quota rather than a judgement about risk.

None of these are coding failures. The code compiled, the tests passed and the reviews found real but ordinary defects. The cost came from building the wrong amount of the right thing.

## What I'm changing

The lessons point at how I prepare work, not at how the loop runs.

- **Write the scope down before the run.** Each story should list what the pilot needs and what is deferred, so the agent can't build past it without contradicting an explicit line.
- **Name tests by risk instead of counting them.** The backend instructions now say not to test CRUD round-trips or simple 404s. Tests that protect tenancy are different. A cross-organisation request must return 404, and a non-member must be forbidden on every scheme route.
- **Check the plan against `dev` first.** A plan written a week ago can describe code that someone has since merged.

Epic 4 opened with a story that restored the membership and role tests the Epic 3 trim had dropped, and added a payable-currency check. Even that story went through one more round of cutting. It added a 404 test for an unknown ID and then removed it before merge, because the lean-test rule says simple 404s aren't worth a test. The cross-organisation 404s stayed, because they protect tenancy. I think that's the rule working as intended. The line between the two cases is now written down, so the agent and I don't have to argue it from scratch each time.

## Reflection

An unattended loop makes it cheap to produce code and expensive to produce the wrong amount of it. Most of my judgement as the engineer now goes into what happens before the run. I decide what the pilot needs, which tests protect something real, and whether the plan still matches the code. When I did that work up front, the loop's output needed little review. When I skipped it, I spent the review deleting code.
