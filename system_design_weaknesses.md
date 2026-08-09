# System Design Weaknesses
Last updated: 2026-08-09 (session 38 - GitHub Actions)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Cannot derive scale numbers independently | 20 | 2026-08-09 |
| Asserts a traffic model without sanity-checking it | 2 | 2026-08-09 |
| Arithmetic slips in BoE / SLA math | 17 | 2026-08-06 |
| Refuses to quantify when asked directly for a number | 4 | 2026-08-09 |
| Never states what is out of scope | 3 | 2026-08-09 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Vague response shape (no explicit fields) | 23 | 2026-08-09 |
| Only designs read endpoints; omits the write/agent contract | 1 | 2026-08-09 |
| Missing pagination on list endpoints | 7 | 2026-08-08 |
| No idempotency/dedupe on an at-least-once ingress | 1 | 2026-08-09 |
| Never produces an API at all when time-pressured | 1 | 2026-08-06 |

## Deep Dives
| Weakness | Sessions | Last Seen |
|---|---|---|
| Asks for hints / leans on interviewer | 20 | 2026-08-09 |
| Abandons the round rather than attempting an answer | 1 | 2026-08-09 |
| Hand-waves core algorithm (geo, sharding, consensus) | 11 | 2026-08-04 |
| Declines to attempt sizing before being pushed | 5 | 2026-08-09 |
| Doesn't size the dependency or cost he introduces | 3 | 2026-08-04 |

## Architecture & Trade-offs
| Weakness | Sessions | Last Seen |
|---|---|---|
| States choices without naming the alternative | 21 | 2026-08-09 |
| Never reaches a high-level design at all | 1 | 2026-08-09 |
| Weakens own NFR when it conflicts, instead of resolving | 2 | 2026-08-09 |
| Missing resilience patterns (DLQ, breaker, failover) | 13 | 2026-08-06 |
| Names a datastore category, never a concrete system | 2 | 2026-08-08 |

## Communication & Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Requires multiple prompts to fully articulate ideas | 27 | 2026-08-09 |
| Over-runs requirements phase, starves the deep dive | 24 | 2026-08-09 |
| Doesn't volunteer break/fix in deep dives | 18 | 2026-08-08 |
| Answers one part of a multi-part question per turn | 5 | 2026-08-09 |
| Doesn't reach for own production experience as evidence | 2 | 2026-08-09 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Weak | 2026-08-09 |
| Leads with trade-offs vs alternatives | Weak | 2026-08-09 |
| Pushes scale until it breaks | Weak | 2026-08-09 |
| API as a designed contract | Weak | 2026-08-09 |
| Operability / second-order concerns | Mixed | 2026-08-09 |
| Pace (core by mid, deep dive after) | Weak | 2026-08-09 |
