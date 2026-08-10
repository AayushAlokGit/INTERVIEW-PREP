# System Design Weaknesses
Last updated: 2026-08-10 (session 39 - Instagram)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Cannot derive scale numbers independently | 21 | 2026-08-10 |
| Arithmetic slips in BoE / SLA math | 18 | 2026-08-10 |
| Refuses to quantify when asked directly for a number | 5 | 2026-08-10 |
| Asserts a traffic model without sanity-checking it | 3 | 2026-08-10 |
| Never states what is out of scope | 3 | 2026-08-09 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Vague response shape (no explicit fields) | 24 | 2026-08-10 |
| Missing pagination on list endpoints | 7 | 2026-08-08 |
| Misses read/delete endpoints until prompted | 2 | 2026-08-10 |
| Only designs read endpoints; omits the write/agent contract | 1 | 2026-08-09 |
| No idempotency/dedupe on an at-least-once ingress | 1 | 2026-08-09 |

## Deep Dives
| Weakness | Sessions | Last Seen |
|---|---|---|
| Asks for hints / leans on interviewer | 21 | 2026-08-10 |
| Hand-waves core algorithm (geo, sharding, consensus) | 11 | 2026-08-04 |
| Declines to attempt sizing before being pushed | 5 | 2026-08-09 |
| Leaves the final scale-break unsolved | 2 | 2026-08-10 |
| Doesn't size the dependency or cost he introduces | 4 | 2026-08-10 |

## Architecture & Trade-offs
| Weakness | Sessions | Last Seen |
|---|---|---|
| States choices without naming the alternative | 22 | 2026-08-10 |
| Missing resilience patterns (DLQ, breaker, failover) | 13 | 2026-08-06 |
| Omits an entire subsystem the FRs require (CDN, transcode) | 1 | 2026-08-10 |
| Weakens own NFR when it conflicts, instead of resolving | 2 | 2026-08-09 |
| Names a datastore category, never a concrete system | 2 | 2026-08-08 |

## Communication & Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Requires multiple prompts to fully articulate ideas | 28 | 2026-08-10 |
| Over-runs requirements phase, starves the deep dive | 25 | 2026-08-10 |
| Doesn't volunteer break/fix in deep dives | 18 | 2026-08-08 |
| Answers one part of a multi-part question per turn | 6 | 2026-08-10 |
| Doesn't reach for own production experience as evidence | 3 | 2026-08-10 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Mixed | 2026-08-10 |
| Leads with trade-offs vs alternatives | Mixed | 2026-08-10 |
| Pushes scale until it breaks | Weak | 2026-08-10 |
| API as a designed contract | Strong | 2026-08-10 |
| Operability / second-order concerns | Mixed | 2026-08-10 |
| Pace (core by mid, deep dive after) | Weak | 2026-08-10 |
