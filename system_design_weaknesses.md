# System Design Weaknesses
Last updated: 2026-08-22 (session 49 — system design round: LLM Serving Platform)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Arithmetic slips in BoE / SLA math | 26 | 2026-08-20 |
| Asserts a traffic model without sanity-checking it | 14 | 2026-08-20 |
| Produces zero NFRs / no numbers at all | 1 | 2026-08-22 |
| Never asks for scale givens when invited | 1 | 2026-08-22 |
| Stops one step short of the number that decides it | 3 | 2026-08-19 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Response omits load-bearing fields (cursor, id, status) | 33 | 2026-08-19 |
| In-scope FR ships with no endpoint or mechanism | 9 | 2026-08-17 |
| No error/status semantics on any endpoint | 8 | 2026-08-19 |
| No idempotency key on a retryable write | 3 | 2026-08-20 |
| No position on cursor stability under concurrent writes | 1 | 2026-08-20 |

## Deep Dives
| Weakness | Sessions | Last Seen |
|---|---|---|
| Asks for hints / leans on interviewer | 22 | 2026-08-13 |
| Hand-waves core algorithm (geo, sharding, consensus) | 12 | 2026-08-13 |
| Declines to attempt sizing before being pushed | 6 | 2026-08-19 |
| Moves on from a break he just identified | 1 | 2026-08-20 |
| Names a mitigation instead of resolving the break | 1 | 2026-08-19 |

## Architecture & Trade-offs
| Weakness | Sessions | Last Seen |
|---|---|---|
| States choices without naming the alternative | 25 | 2026-08-20 |
| Missing resilience patterns (DLQ, breaker, failover) | 15 | 2026-08-20 |
| Names a datastore category, never a concrete system | 3 | 2026-08-19 |
| Trade-offs on own inventions, none on dependencies | 1 | 2026-08-20 |
| Makes a scoping decision silently instead of stating it | 1 | 2026-08-20 |

## Communication & Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Over-runs requirements phase, starves the deep dive | 35 | 2026-08-22 |
| FRs restate the prompt instead of making choices | 1 | 2026-08-22 |
| Abandons the round mid-phase instead of pushing through | 1 | 2026-08-22 |
| Doesn't volunteer break/fix in deep dives | 18 | 2026-08-08 |
| Never states what is out of scope | 1 | 2026-08-22 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Weak — two well-chosen clarifying questions up front (what "better service" means; whether provisioning + weight-loading are owned), then four FRs of which three restate the prompt verbatim. No trap self-raised on a problem that hands over several for free: cancellation of an in-flight generation, reconnect of a dropped stream, KV-cache statefulness, what happens to a half-streamed response when a GPU dies | 2026-08-22 |
| Leads with trade-offs vs alternatives | Not observed — no design produced. Prior sitting: Mixed | 2026-08-22 |
| Pushes scale until it breaks | Weak — nothing sized, no givens requested despite two invitations. On this problem the capacity model IS the interview: QPS is the wrong unit, GPU capacity is tokens/sec, and requests differ ~100x in cost. That derivation was never attempted. Prior sitting: Strong (post search) | 2026-08-22 |
| API as a designed contract | Not observed — phase never reached. Prior sitting: Strong | 2026-08-22 |
| Operability / second-order concerns | Not observed — phase never reached. Prior sitting: Weak | 2026-08-22 |
| Pace (core by mid, deep dive after) | Weak — 15 minutes to produce an incomplete requirements phase (FRs only, no out-of-scope, no NFRs), then the round was abandoned at minute 29 with nothing past requirements reached. This reverses the prior sitting's 22-minute complete round; the Strong pace read from 2026-08-20 remains unreproduced | 2026-08-22 |
