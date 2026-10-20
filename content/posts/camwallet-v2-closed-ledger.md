+++
authors = ['Radin']
title = 'CamWallet v2: A Closed Ledger With No Bank in the Path'
date = '2026-10-20'
summary = "How CamWallet's second version changed two assumptions, who decides eligibility and where payment happens, and why everything else in the design follows from those two changes."
ShowToc = true
TocOpen = false
draft = false
tags = ['camwallet', 'tsc', 'ledger', 'payments', 'spring-boot', 'postgres']
+++

## What CamWallet is

CamWallet is a platform at TSC for government benefit schemes. A scheme owner, usually a ministry or agency, funds a programme and decides who receives value. Beneficiaries spend that value with approved merchants. CamWallet holds the value, checks each payment when it happens, and keeps a record good enough to reconcile and audit.

There are three surfaces. Scheme owners and merchants use a Back Office portal built with Nuxt 4. Beneficiaries pay through the CamDigiKey app, which is Cambodia's digital identity app. Merchants accept payments in a native Flutter Merchant App. The backend is Java 25 on Spring Boot 4.1 with Spring Modulith, Spring Data JDBC and Postgres.

## v2 is two changes, not a rewrite

Version 1 worked like this.

1. A scheme defined eligibility rules, and CamWallet decided who qualified.
2. People who qualified received value in a wallet.
3. They paid merchants through a bank, using Bakong, ABA or ACLEDA.
4. The bank confirmed the payment, and CamWallet recorded the outcome.

Version 2 changes the first and third steps. The scheme owner now decides who qualifies in their own system, and CamWallet records that decision without re-checking it. Payment now happens inside CamWallet, between a beneficiary's value and a merchant's balance, with no bank in the path.

Everything else carries over, including the double-entry ledger, CamDigiKey login, disbursement scheduling, idempotency and the audit trail. That is why we treated v2 as a milestone on the existing codebase rather than a new system. When we wrote down the full v1-to-v2 delta, it came to ten rows: four additions, three deletions, two renames and one re-point.

## Why the two changes cascade

Each change removes a lot of machinery.

Once the scheme owner decides eligibility, CamWallet doesn't need eligibility attributes, a registry lookup or an approval workflow. The `schemes/eligibility/` module goes away. Eligibility evaluation, merchant vetting and base-registry integration are now permanently out of scope.

Once payment is internal, merchants need a balance, so each merchant gets a ledger account. The money is owed to the merchant until the scheme owner pays them outside the system, so settlement has to debit that balance. The payment-gateway clients for the banks have nothing left to do and are deleted.

The design rests on one sentence. A scheme owner funds a programme and decides who qualifies, and CamWallet makes sure the money is only spent with approved merchants and then tells the owner exactly what to pay each merchant.

## The ledger

Everything is a transfer between two accounts. Entries are appended and never changed, and every balance can be explained by replaying its entries. Reconciliation becomes arithmetic instead of investigation.

| Account | What it holds |
|---|---|
| `scheme:{id}:expenditure` | Total ever granted in the scheme |
| `grant:{id}` | One grant's unspent value |
| `payable:{scheme}:{merchant}` | What the scheme owes one merchant |
| `scheme:{id}:paid_out` | Total recorded as paid to merchants |
| `scheme:{id}:returned` | Unspent value from expired grants |

Each enrolment is a **grant** with its own amount and validity dates, and a beneficiary can hold several. A wallet is not an account. It is the set of a beneficiary's grant accounts in one scheme, and its spendable balance only counts grants that are valid today.

Payables are per scheme. A merchant serving two schemes has two payables, because each scheme settles separately and the obligations must not pool.

| Operation | From → To |
|---|---|
| Disburse | scheme expenditure → grant |
| Pay | grant (soonest end date first) → payable |
| Record payout | payable → scheme paid_out |
| Return expired value | grant → scheme returned |

A payment that draws on two grants writes two debits and one credit in one transaction. Because expenditure moves only on disbursement, reconciliation is a single identity per scheme that we can check in one query and assert in a test.

```
granted = Σ grant balances + Σ payables + paid_out + returned
```

## Making it safe in Postgres

Removing the bank removed the distributed transaction. A payment debits grant accounts and credits a payable in one ACID transaction inside one database. There is no two-phase commit and no compensating action. That single fact made the rest of the design much simpler.

A few rules keep the ledger honest.

- **Entries are immutable.** A database trigger rejects `UPDATE` and `DELETE` on `ledger_entries`. A correction is a new reversing entry.
- **Cached balances change atomically.** Grant and payable accounts cache their balance, because payment checks read those two numbers. The cache only changes inside one posting method, as `SET balance = balance + :delta`. A read-modify-write in Java isn't allowed, because many wallets credit the same payable at once. A test asserts that each cached balance equals the sum of its entries.
- **Every transfer balances.** The posting code rejects an unbalanced set of entries before it writes anything.
- **Locks guard the checks, not the arithmetic.** A payment locks the beneficiary's wallet row before it reads any grant. A payment from a dynamic QR locks the QR first, so one QR can't be paid twice. Credits need no lock, because the atomic delta already makes them safe.
- **Money is exact decimal.** Amounts are `numeric(19,4)` and always travel with their currency. A posting across two currencies is rejected.

## Declines have a closed vocabulary

Every declined payment carries exactly one reason code from a fixed list, evaluated in a fixed order. The list includes `QR_REVOKED`, `QR_ALREADY_PAID`, `QR_EXPIRED`, `MERCHANT_INACTIVE`, `NO_SCHEME_AT_MERCHANT`, `MERCHANT_NOT_PARTICIPATING`, `CREDIT_NOT_YET_VALID`, `CREDIT_EXPIRED` and `INSUFFICIENT_BALANCE`.

Adding a code later is cheap. Rewriting months of free-text rejection messages into something a report can group is not.

## What it removed from the build

The original 2026 proposal named Postgres, MinIO and Kafka. For a four-week build at pilot volume, Postgres covers most of that list. It holds the ledger, configuration, audit trail and Spring Modulith's event publication registry. Each piece of infrastructure we didn't add is one fewer thing to deploy, monitor and explain to the next developer.

## Merchant QRs

A recent piece of work let merchants manage their own static QRs. A merchant often takes payments at more than one counter, and each counter needs its own printed QR with a label so staff and payers know where it belongs. A QR names the merchant, currency and label but no scheme, and the beneficiary picks the scheme when they pay.

The EMV QR format limits the merchant name in tag 59 to 25 characters. We had been filling it from the merchant's English name, which had no length limit. Merchants now get a separate display name capped at 25 printable ASCII characters, and that name is what appears on the QR.

## Looking back

The most useful document in the project is one page called "What v2 actually is." It lists the four steps, marks the two that changed, and traces every other decision back to them. New readers start there. When a design discussion drifted, we went back to that page and asked which of the two changes the proposal followed from. If it followed from neither, it usually belonged in a later phase.
