+++
authors = ['Radin']
title = 'CamWallet v2: Benefit Payments on a Closed Ledger'
date = '2026-10-20'
summary = "How CamWallet v2 lets government schemes pay beneficiaries and merchants in real time without a bank integration, why a double-entry ledger makes it auditable, and what building it with an AI agent loop taught me."
ShowToc = true
TocOpen = false
draft = false
tags = ['camwallet', 'tsc', 'govtech', 'ledger', 'payments', 'ai-assisted-development']
+++

## Introduction

I'm a software engineer at TSC (Techo Startup Center), and since September 2026 I've been building **CamWallet v2**. CamWallet is a platform that helps government benefit schemes deliver value to the people they're meant for, and lets those people spend it with approved merchants.

This post explains what v2 changes and why it helps scheme owners, beneficiaries and merchants. It walks through the ledger design that makes the money auditable, and it ends with what I learned from building the epics with an unattended AI agent loop. Engineers working on payments or wallets should get the most from the ledger section. Teams trying AI-assisted development should get the most from the last two sections.

## What v2 changes, and why it matters

Version 1 worked in four steps.

1. A scheme defined eligibility rules, and CamWallet decided who qualified.
2. People who qualified received value in a wallet.
3. They paid merchants through a bank, using Bakong, ABA or ACLEDA.
4. The bank confirmed the payment, and CamWallet recorded the outcome.

Version 2 changes the first and third steps. The scheme owner decides who qualifies in their own system, and CamWallet records that decision without re-checking it. Payment happens inside CamWallet, from a beneficiary's grant to a merchant's balance, with no bank in the path. The ledger, CamDigiKey login, disbursement scheduling, idempotency and audit trail all carry over, so v2 is a milestone on the existing codebase rather than a rewrite.

Those two changes give each group something concrete.

- **Scheme owners** keep control of eligibility, which is their policy decision anyway. CamWallet makes sure the money is spent only with approved merchants, and it tells the owner exactly what to pay each merchant at settlement.
- **Beneficiaries** pay by scanning a merchant's QR in the CamDigiKey app, the national digital identity app they already use. The payment completes immediately, and a declined payment always records one specific reason.
- **Merchants** see each payment arrive in the Merchant App and know exactly what each scheme owes them. A merchant with several counters can print a labelled QR for each one.
- **Auditors and finance teams** can reconcile every scheme with one balance check instead of investigating transactions by hand.
- **The project** delivers a working pilot without waiting for a bank integration. Removing the bank also removed the payment-gateway clients, the eligibility engine and the registry lookup. For a four-week build at pilot volume, Postgres replaced most of the Postgres, MinIO and Kafka stack in the original proposal.

## The ledger

Everything is a transfer between two accounts. Entries are appended and never changed, and replaying an account's entries explains its balance. That turns reconciliation into arithmetic.

| Account | What it holds |
|---|---|
| `scheme:{id}:expenditure` | Total ever granted in the scheme |
| `grant:{id}` | One grant's unspent value |
| `payable:{scheme}:{merchant}` | What the scheme owes one merchant |
| `scheme:{id}:paid_out` | Total recorded as paid to merchants |
| `scheme:{id}:returned` | Unspent value from expired grants |

Each enrolment is a **grant** with its own amount and validity dates, and a beneficiary can hold several. A wallet isn't an account. It is the set of a beneficiary's grants in one scheme, and its spendable balance counts only grants that are valid today. Payables are kept per scheme, so a merchant serving two schemes has two payables that settle separately.

| Operation | From → To |
|---|---|
| Disburse | scheme expenditure → grant |
| Pay | grant (soonest end date first) → payable |
| Record payout | payable → scheme paid_out |
| Return expired value | grant → scheme returned |

Expenditure moves only on disbursement, so each scheme satisfies one identity that we check in a single query and assert in a test.

```
granted = Σ grant balances + Σ payables + paid_out + returned
```

## Keeping it correct in Postgres

With no bank involved, a payment is one ACID transaction in one database. It needs no distributed transaction and no compensating action. A few rules protect the ledger.

- **Entries are immutable.** A trigger rejects `UPDATE` and `DELETE` on ledger entries, so a correction is always a new reversing entry.
- **Cached balances change atomically.** Grant and payable balances are cached for payment checks. They change only inside one posting method, through `SET balance = balance + :delta`, never through a read-modify-write in Java. A test asserts that each cached balance equals the sum of its entries.
- **Every transfer balances.** The posting code rejects an unbalanced set of entries before writing.
- **Locks guard the checks.** A payment locks the beneficiary's wallet row before reading any grant, and a dynamic QR is locked first so it can't be paid twice.
- **Money is exact.** Amounts are `numeric(19,4)` and always carry a currency, and a posting across two currencies is rejected.

Each declined payment carries exactly one code from a fixed list, such as `QR_EXPIRED`, `MERCHANT_NOT_PARTICIPATING`, `CREDIT_EXPIRED` or `INSUFFICIENT_BALANCE`. Reports can group declines by reason from day one, and adding a code later is cheap.

## Building it with an AI agent loop

I implemented the v2 epics with the BMad Method and bmad-loop, a CLI that runs the implementation phase unattended. It picks a story, writes a spec, implements it, runs a compile-and-typecheck gate and moves on. I held retrospectives after Epic 1 (merchant management) and Epic 3 (scheme participation), and both found the same pattern.

In Epic 1 the loop built all seven stories, plus seven capabilities the pilot didn't need, including member management, server-side paging and an idempotency key. Human review removed them. The feature commit was about 2,500 lines, and the net change after the slim-down was about 550. The loop's own review step never triggered, so every defect was found by a person.

Epic 3 was smaller and still left about 250 lines on `dev` that didn't survive. The plan said to build a JSON deserializer that `dev` already had, and the agent followed the plan. A rule to write minimal tests was overshot at first, and then the trim cut the only tests that checked membership and roles.

The code compiled and the tests passed in both epics. The cost came from building the wrong amount of the right thing. The loop is good at matching a plan, but it doesn't notice when the plan has gone stale or asks for more than the pilot needs.

## What you can take from this

These are the lessons I'd pass on to anyone building something similar.

- **Look for the one or two assumptions that drive the design.** Writing v2 down as two changed steps out of four made every later decision easier to test. If a proposal followed from neither change, it usually belonged in a later phase.
- **Keep money movements inside one database when you can.** A single transaction removes the hardest failure cases in payments.
- **Make the ledger append-only and prove it balances.** An immutable entry table plus one reconciliation identity gives auditors a check they can run themselves.
- **Use a closed list of decline reasons.** It costs nothing at the start and saves a painful clean-up later.
- **With an agent loop, do the judgement work before the run.** Write down each story's pilot scope and deferrals, check the plan against the current branch, and name the tests that protect something real, such as tenancy and roles, instead of setting a test count.
- **Keep a human review gate on purpose.** In both epics, every real defect was found in human review.
