# System Design Weaknesses
Last updated: 2026-10-01 (session 52 — system design round: Robinhood)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Arithmetic slips in BoE / SLA math | 29 | 2026-10-01 |
| Asserts a traffic model without sanity-checking it | 17 | 2026-10-01 |
| Stops one step short of the number that decides it | 6 | 2026-10-01 |
| Produces zero NFRs / no numbers at all | 1 | 2026-08-22 |
| Never asks for scale givens when invited | 1 | 2026-08-22 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Response omits load-bearing fields (cursor, id, status) | 34 | 2026-09-30 |
| In-scope FR ships with no endpoint or mechanism | 12 | 2026-10-01 |
| No error/status semantics on any endpoint | 11 | 2026-10-01 |
| No idempotency key on a retryable write | 4 | 2026-09-30 |
| No position on cursor stability under concurrent writes | 1 | 2026-08-20 |

## Deep Dives
| Weakness | Sessions | Last Seen |
|---|---|---|
| Asks for hints / leans on interviewer | 23 | 2026-09-29 |
| Hand-waves core algorithm (geo, sharding, consensus) | 15 | 2026-10-01 |
| Declines to attempt sizing before being pushed | 9 | 2026-10-01 |
| Names a mitigation instead of resolving the break | 3 | 2026-09-30 |
| Declares a scale break solved without sizing it | 1 | 2026-10-01 |

## Architecture & Trade-offs
| Weakness | Sessions | Last Seen |
|---|---|---|
| States choices without naming the alternative | 28 | 2026-10-01 |
| Missing resilience patterns (DLQ, breaker, failover) | 18 | 2026-10-01 |
| Names a datastore category, never a concrete system | 3 | 2026-08-19 |
| Calls external system before persisting own record | 1 | 2026-10-01 |
| Entity lacks id / owner needed by its own FRs | 1 | 2026-10-01 |

## Communication & Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Over-runs requirements phase, starves the deep dive | 38 | 2026-10-01 |
| Doesn't volunteer break/fix in deep dives | 21 | 2026-10-01 |
| FRs restate the prompt instead of making choices | 4 | 2026-10-01 |
| Never states what is out of scope | 3 | 2026-09-30 |
| Abandons the round mid-phase instead of pushing through | 3 | 2026-10-01 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Mixed — self-raised the exchange status-query question, Idempotency-Key, pub/sub at-most-once with a fallback, and out-of-scope; never raised the exchange-then-DB crash, the timeout, cancel-vs-fill or order-status push; ended the round when the deep dive got specific | 2026-10-01 |
| Leads with trade-offs vs alternatives | Mixed — Cassandra vs SQL for prices and SQL for orders each justified; no alternative named for SSE, Redis pub/sub, polling, or the write ordering | 2026-10-01 |
| Pushes scale until it breaks | Weak — declared the 1B/s fan-out "largely reduced" without sizing an instance; asserted PostgreSQL would cope | 2026-10-01 |
| API as a designed contract | Mixed — Idempotency-Key, cursor + nextCursor, order id first pass; no order type/price, no order-status push, no error semantics | 2026-10-01 |
| Operability / second-order concerns | Weak — no reconciliation, feed disconnect, market-open handling, or any metric raised | 2026-10-01 |
| Pace (core by mid, deep dive after) | Weak — requirements 19 vs 8, HLD done at 45, deep dive abandoned at 56; ~10 min round-trip tax | 2026-10-01 |
