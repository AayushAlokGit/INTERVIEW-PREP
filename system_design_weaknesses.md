# System Design Weaknesses
Last updated: 2026-08-13 (session 41 - design sprint: Ticketmaster, Ad Click Aggregator)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Cannot derive scale numbers independently | 21 | 2026-08-10 |
| Arithmetic slips in BoE / SLA math | 21 | 2026-08-13 |
| Asserts a traffic model without sanity-checking it | 6 | 2026-08-13 |
| Refuses to quantify when asked directly for a number | 6 | 2026-08-13 |
| Omits read:write ratio, storage growth, durability | 2 | 2026-08-13 |
| Defers a sizing question and never returns to it | 1 | 2026-08-13 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Vague response shape (no explicit fields) | 27 | 2026-08-13 |
| Misses read/delete endpoints until prompted | 5 | 2026-08-13 |
| No idempotency on the most contended write | 3 | 2026-08-13 |
| No error/status semantics on any endpoint | 2 | 2026-08-13 |
| Missing the endpoint the core flow needs (e.g. hold) | 1 | 2026-08-13 |

## Deep Dives
| Weakness | Sessions | Last Seen |
|---|---|---|
| Asks for hints / leans on interviewer | 22 | 2026-08-13 |
| Hand-waves core algorithm (geo, sharding, consensus) | 12 | 2026-08-13 |
| Declines to attempt sizing before being pushed | 5 | 2026-08-09 |
| Ends the round before the deep dive is reached | 1 | 2026-08-13 |
| Doesn't size the dependency or cost he introduces | 4 | 2026-08-10 |

## Architecture & Trade-offs
| Weakness | Sessions | Last Seen |
|---|---|---|
| States choices without naming the alternative | 23 | 2026-08-13 |
| Missing resilience patterns (DLQ, breaker, failover) | 13 | 2026-08-06 |
| Names a datastore without its key/partition design | 1 | 2026-08-13 |
| Weakens own NFR when it conflicts, instead of resolving | 2 | 2026-08-09 |
| Names a datastore category, never a concrete system | 2 | 2026-08-08 |

## Communication & Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Over-runs requirements phase, starves the deep dive | 28 | 2026-08-13 |
| Requires multiple prompts to fully articulate ideas | 29 | 2026-08-13 |
| Doesn't volunteer break/fix in deep dives | 18 | 2026-08-08 |
| Answers ~70% of a question, rest costs a round-trip | 8 | 2026-08-13 |
| Spends requirements budget on clarifying questions | 1 | 2026-08-13 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Weak | 2026-08-13 |
| Leads with trade-offs vs alternatives | Weak | 2026-08-13 |
| Pushes scale until it breaks | Weak | 2026-08-13 |
| API as a designed contract | Weak — pagination/idempotency now habitual, but response shapes still have no named fields | 2026-08-13 |
| Operability / second-order concerns | Not observed | 2026-08-13 |
| Pace (core by mid, deep dive after) | Weak — 2/2 requirements gates blown; entities and API both land early | 2026-08-13 |
