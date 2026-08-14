# Design Sprint Transcript (front half, timeboxed)
**Date:** 2026-08-13
**Start Time:** 19:46:30 · **End Time:** 20:02:36
**Problem:** Payment System (marketplace checkout → PSP charge → ledger → seller payout)
**Difficulty:** 4/5 — exactly-once money movement across an unreliable external PSP, a double-entry ledger with a balance invariant, an async charge path, and a second money flow (payouts) on a different consistency posture.
**Front-half readiness: 1/5**
**Front half complete inside 17:00: no** — only Requirements existed. Core entities and API design were never started; the drill was ended by request at 16:06 with two of three phases empty.

| Phase | Pace target | Landed at | ± vs target | Messages used | Score |
|---|---|---|---|---|---|
| Requirements | 8:00 | 12:14 | +4:14 | 4 (3 clarifying, 1 content) | 2/5 |
| Core entities | 12:00 | never | — | 0 | 0/5 |
| API design | 17:00 | never | — | 0 | 0/5 |

**Budget allocation:** 12:14 requirements / 0:00 entities / 0:00 API, then stopped at 16:06. Of the 12:14 spent on requirements, **10:19 was clarifying questions with zero content written**. The content message itself took 1:55 to produce. This is not a thinking-speed problem — writing was fast once it started. The entire box was consumed before writing began.

**First-pass completeness:** The single requirements content message was one-pass and coherent, but ~60% complete against the 7-item walk: no read:write ratio, no storage growth, no latency percentile, out-of-scope limited to one item, availability stated globally rather than per-path. Nothing was back-filled because the drill ended. The three clarifying questions were serial, each costing a full round-trip:
- 2:08 — "does the system facilitate the payment or just record it?" — legitimate scope fork, correctly asked.
- 7:44 — "what scale should the system be designed to handle?" — legitimate, but arriving 7:44 in. Batched with the first question it would have cost 2:00 instead of 7:44.
- 10:19 — "what does a payment ledger entry look like?" — not a clarifying question. That is the core-entities phase, asked of the interviewer.

**Plausibility check:** Not performed, and it had two live targets. (a) **10KB per ledger entry** — a ledger row is ~6 columns (id, txn_id, account, direction, amount, currency, timestamp) ≈ 200–500 bytes. He is 20–50× high, which inflates the storage figure by the same factor, and storage was the number the explicitly-supplied 7-year retention was inviting him to derive. (b) **Unit slip** — "2 Mb/s at peak 10Mb/s" for 200 rows/s × 10KB = 2 **MB**/s, 10 **MB**/s at peak. Megabit vs megabyte, 8×, in a payments design. One line — "10KB for a row that's basically six columns? call it 500B" — catches the first and forces a re-derive that would have caught the second. Caps Requirements at 3 independently; the missing NFR content puts it at 2.

## What he produced (verbatim, as it stood at 16:06)

### Requirements
```
FRs ->
1. Allow buyer to make payments by triggering PSPs.
2. Record the payment made by buyer.
3. On successful payment by buyer , schedule the seller to be paid.
4. Each payment made must be recorded in a ledger exposed for reconciliation.
5. Display payment status of each payment made.

Out of scope ->
1. Frau detection

NFRs ->
1. Highly available for accepting payments (99.99~52mins downtime/year)
2. Handle PSP timeouts and make idempotent calls to PSP to avoid double charging.
3. Durably store each payment record.
4. Strong consistency needed for payment status data.
5. 50M DAU , with 0.2 checkouts/user/day -> 100 payments/s peak 5x. Each payment creates 2 ledge entries
200 ledger entires/s, peak 5x ,assuming each ledger entry 10Kb -> 2 Mb/s at peak 10Mb/s
```

### Core entities
*Never started.*

### API design
*Never started.*

## What was still missing at 17:00

**Requirements:**
- Read:write ratio — absent. Payments is read-heavy (status polling, order history, seller dashboards, reconciliation scans); the ratio drives the datastore choice.
- Storage growth — absent, despite 7-year retention being handed to him explicitly. The most obviously-invited derivation in the problem.
- Latency NFR with a percentile — absent. "Highly available" is not latency. The interesting answer here is that the PSP round-trip is 1–3s, which forces the charge to be async and shapes the entire API.
- Out-of-scope — one item (fraud). Missing: chargebacks/disputes, multi-currency FX, tax, PCI/card storage, subscriptions, split payments.
- Availability stated once, globally. The senior answer is asymmetric: charge path 99.99% (lost revenue per minute), payout path 99.9% (a daily batch tolerates hours).
- Consistency stated as one line for "payment status data" rather than per subsystem (ledger vs payment state machine vs seller balance vs status display).

**Everything in core entities and API design**, including the two items that carry this problem: the `UNIQUE (buyer_id, idempotency_key)` constraint where "no double-charge" actually lives, and `Idempotency-Key` as a required header on every mutating endpoint.

## Where the time went

10:19 of a 17:00 box in question-asking mode before a single line of content. The third question ("what does a ledger entry look like?") is the diagnostic one — it asks the interviewer to supply a design decision that is itself being graded, and it cost 2:35 to receive a one-line refusal. Content, once started, took 1:55 for a five-FR / five-NFR block. The capability is there; the trigger to start writing is not.

Positive deltas worth recording: idempotency named as an NFR unprompted (first time), payout listed as an FR (most candidates forget the money leaving), consistency scoped to a named subsystem rather than asserted globally.

## Ideal front half (writable in the same 17 minutes)

### Requirements — target 7:00
```
Clarify (ONE message, up front):
  - Do we initiate the charge or only record it? → initiate.
  - Give me DAU, checkout rate, order value, retention, peak.

FRs
1. Buyer checks out → we charge their card via PSP, exactly once.
2. We record every money movement in an immutable double-entry ledger.
3. Buyer can poll payment status (pending/authorized/captured/failed/refunded).
4. Refunds: full or partial, against a captured payment.
5. Sellers are paid out on a daily batch from their ledger balance.
6. Ledger is queryable for reconciliation against the PSP settlement file.

Out of scope
  fraud/risk scoring · chargebacks & disputes · multi-currency FX · tax
  · subscriptions/recurring · split payments · card storage (PSP tokenizes,
    we never see a PAN — PCI stays outside our boundary)

NFRs
  Availability — charge path 99.99% (52 min/yr); a failed checkout is lost
    revenue. Payout path 99.9% is fine — daily batch, hours of delay invisible.
  Consistency — payment state machine + ledger: strongly consistent, single
    transaction. No double-charge, no unbalanced ledger. Seller payout
    balance: eventually consistent, derived from ledger. Buyer status
    display: read-your-writes.
  Latency — checkout p50 < 200ms excluding PSP, p99 < 1s. PSP round-trip is
    1–3s so the charge is ASYNC: return payment_id immediately, client polls.
    Status read p99 < 100ms.
  Durability — ledger is system of record: 11 nines, replicated, WAL, never
    hard-deleted. Everything else is reconstructible from it.

Traffic model
  50M DAU × 0.2 checkouts/day = 10M payments/day
  10M / 10^5 s = 100 payments/s avg; peak 5× = 500/s
  Double-entry: 2 rows/payment, +2 per refund (2%) ≈ 2.04 → 2
  → 200 ledger writes/s avg, 1000/s peak
  Reads: ~3 status polls per payment + order history + seller dashboards
  → ~20:1 read:write. Read-heavy; the writes are the correctness-critical half.

  PLAUSIBILITY CHECK: 500 payments/s × $50 = $25k/s at peak = $2.2B/day gross.
  Amazon-scale — plausible for "large marketplace", and it says one minute of
  charge-path downtime costs ~$1.5M. That defends the 99.99%. It also says a
  single PSP is a single point of failure — I'd want two.

Storage
  A ledger row is ~6 columns + metadata ≈ 500 B (a ROW, not a document —
  10KB would be 20× too high).
  200 rows/s × 10^5 s = 20M rows/day × 500 B = 10 GB/day
  × 365 × 7 yr ≈ 25 TB → partitioned Postgres/Spanner, nothing exotic.
  Peak bandwidth 1000 rows/s × 500 B = 500 KB/s — trivial.
```
**What this buys:** fills the four NFR slots he left empty (read:write, storage growth, latency percentile, per-subsystem consistency), and the plausibility line does the work skipped twice — it catches the 10KB row *and* converts the traffic number into the business fact ($1.5M/min) that justifies the availability target, turning a recited SLA into a defended one.

### Core entities — target 11:00
```
Payment                       -- buyer intent + PSP outcome
  payment_id (PK, uuid)
  idempotency_key   UNIQUE (buyer_id, idempotency_key)  ← anti-double-charge
  order_id, buyer_id, seller_id
  amount_minor (int64 cents — never a float), currency
  status: CREATED → AUTHORIZED → CAPTURED → {REFUNDED, PARTIALLY_REFUNDED}
          CREATED → FAILED
  psp_name, psp_payment_id (null until PSP responds), psp_idempotency_key
  version (int, optimistic concurrency on transitions)
  created_at, updated_at

LedgerEntry                   -- immutable, append-only, double-entry
  entry_id (PK), transaction_id   ← groups debit+credit; they must balance
  account_id, direction (DEBIT|CREDIT), amount_minor, currency
  reference_type (PAYMENT|REFUND|PAYOUT|FEE), reference_id, created_at
  INVARIANT: SUM(credits) − SUM(debits) = 0 per transaction_id
  UNIQUE (reference_type, reference_id, account_id, direction)
    ← makes ledger writes idempotent on retry

Account
  account_id (PK), owner_type (BUYER|SELLER|PLATFORM|PSP_CLEARING), owner_id,
  currency
  NO balance column — balance is derived from the ledger. A cached balance
  that drifts from the ledger is the classic payments bug.

Refund
  refund_id (PK), payment_id, amount_minor, reason
  idempotency_key UNIQUE
  status: PENDING → SUCCEEDED | FAILED
  psp_refund_id

Payout
  payout_id (PK), seller_id, period (date), amount_minor
  status: PENDING → IN_TRANSIT → PAID | FAILED
  UNIQUE (seller_id, period)   ← the daily batch can be safely re-run

ENTITIES THAT ONLY APPEAR UNDER LOAD / FAILURE:
  PSPWebhookEvent (psp_event_id as PK) — PSP webhooks are at-least-once and
    arrive out of order; this table dedupes them and is the reason an async
    PSP callback can be trusted at all.
  OutboxEvent — payment state change written in the SAME transaction as the
    Payment row; without it "charge succeeded but the payout job never heard"
    is a real money bug.
  BalanceSnapshot (account_id, as_of_date, balance) — 7 years of ledger means
    computing a seller balance from scratch is a 25TB scan; snapshot nightly,
    replay only the delta.
```
**What this buys:** the `idempotency_key` UNIQUE constraint is where "no double-charge" actually lives — a database constraint, not a sentence in the NFRs, which is where he put it. The deliberate absence of a `balance` column is a defensible choice an interviewer notices. `PSPWebhookEvent` and `BalanceSnapshot` are the entities that only exist once the failure path and the 7-year retention are taken seriously.

### API design — target 17:00
```
POST /v1/payments                       — initiate a charge
  Header: Idempotency-Key: <uuid>       ← REQUIRED; replay returns the original
                                           response, never re-charges
  Body: { order_id, buyer_id, seller_id, amount_minor, currency,
          payment_method_token }         ← token from PSP.js, never a PAN
  201 → { payment_id, status: "CREATED", created_at }
  Async because the PSP round-trip is 1–3s: return immediately, client polls.
  409 if the same Idempotency-Key is in flight with a different body.

GET /v1/payments/{payment_id}
  200 → { payment_id, status, amount_minor, currency, failure_reason?, version }
  status is the enum the client branches on; version for optimistic reads.

GET /v1/payments?buyer_id=&status=&cursor=&limit=50
  200 → { items: [...], next_cursor }
  Cursor not offset: the list is append-heavy and time-ordered, so offset
  pagination skips/repeats rows as new payments land mid-scan.
  Cursor encodes (created_at, payment_id).

POST /v1/payments/{payment_id}/refunds
  Header: Idempotency-Key REQUIRED
  Body: { amount_minor, reason }        — omit amount ⇒ full refund
  201 → { refund_id, status: "PENDING" }
  422 if amount > captured − already_refunded.

DELETE /v1/payments/{payment_id}        — void, only while CREATED/AUTHORIZED
  200 → { payment_id, status: "CANCELLED" }
  409 if already CAPTURED — that's a refund, not a cancel.
  Payments are never hard-deleted; DELETE means void.

POST /v1/psp/webhooks/{psp_name}        — PSP → us, at-least-once
  Signature verified; dedupe on psp_event_id.
  ALWAYS 200 on duplicate — a non-2xx makes the PSP retry forever.

GET /v1/ledger/entries?account_id=&from=&to=&cursor=&limit=1000
  200 → { items: [...], next_cursor }
  from/to MANDATORY, max 31-day window — 7 years of entries for a
  high-volume seller is millions of rows; an unbounded list here is an
  outage. Full-history export is a job, not a synchronous read.

GET /v1/accounts/{account_id}/balance
  200 → { account_id, balance_minor, currency, as_of }
  as_of is load-bearing: balance = snapshot + delta, so the client must
  know how fresh it is.

GET /v1/payouts?seller_id=&cursor=  → { items, next_cursor }
GET /v1/payouts/{payout_id}         → { payout_id, status, amount_minor,
                                        period, failed_reason? }

Errors: 400 malformed · 401 · 404 · 409 idempotency conflict / illegal state
transition · 422 business rule (over-refund, insufficient balance) · 429 with
Retry-After · 503 PSP unavailable, retryable.
Every 4xx/5xx: { error_code, message, retryable: bool }
  — retryable is the field the client actually branches on.
```
**What this buys:** the entire phase, which did not exist — but specifically the three items dropped in five consecutive sprints. `Idempotency-Key` as a header on every mutating endpoint rather than a sentence in the NFRs. `next_cursor` named in every list response with the cursor-vs-offset reason in one clause. And a stated position on payload size (the mandatory bounded ledger window), which is the "I thought about what this endpoint does when it meets a real seller's 7 years of history" signal that separates a written API from a narrated one.

## Feedback given

**The one habit to change:** ask every clarifying question you have in your FIRST message, then start writing. Three questions asked serially cost 10:19 of a 17:00 box; batched they cost 2:00 and all three phases fit. If a question occurs after you've started writing, write the assumption inline (`assuming 7-year retention`) and keep going — never spend a round-trip on it.

Secondary: "what does X look like?" is never a clarifying question when X is one of the three graded artifacts. If you're asking the interviewer to describe an entity, you are asking them to do the phase you're being scored on.

Content-level: the requirements block was directionally good and the idempotency-as-NFR call was new and correct — but it must become a UNIQUE constraint on a table and a required header on an endpoint before it counts. Stating the intent is the cheap half.
