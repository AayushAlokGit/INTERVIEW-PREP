# System Design Weaknesses
Last updated: 2026-08-08 (session 37 - Online Auction System)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Cannot derive scale numbers independently | 19 | 2026-08-08 |
| Arithmetic slips in BoE / SLA math | 17 | 2026-08-06 |
| Refuses to quantify when asked directly for a number | 3 | 2026-08-08 |
| Never states what is out of scope | 2 | 2026-08-06 |
| Asserts a traffic model without sanity-checking it | 1 | 2026-08-08 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Vague response shape (no explicit fields) | 22 | 2026-08-08 |
| Missing pagination on list endpoints | 7 | 2026-08-08 |
| Never produces an API at all when time-pressured | 1 | 2026-08-06 |
| Names an endpoint without specifying its shape | 2 | 2026-07-29 |
| Asserts a semantic choice without defending it | 1 | 2026-08-08 |

## Deep Dives
| Weakness | Sessions | Last Seen |
|---|---|---|
| Asks for hints / leans on interviewer | 19 | 2026-08-06 |
| Hand-waves core algorithm (geo, sharding, consensus) | 11 | 2026-08-04 |
| Declines to attempt sizing before being pushed | 4 | 2026-08-08 |
| Doesn't size the dependency or cost he introduces | 3 | 2026-08-04 |
| Stops at first fix; no trade-offs on the new design | 2 | 2026-07-29 |

## Architecture & Trade-offs
| Weakness | Sessions | Last Seen |
|---|---|---|
| States choices without naming the alternative | 20 | 2026-08-08 |
| Missing resilience patterns (DLQ, breaker, failover) | 13 | 2026-08-06 |
| Abandons own NFR when a component fails, unreconciled | 1 | 2026-08-08 |
| Doesn't self-catch hot-key/partition in own design | 5 | 2026-08-08 |
| Names a datastore category, never a concrete system | 2 | 2026-08-08 |

## Communication & Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Requires multiple prompts to fully articulate ideas | 26 | 2026-08-08 |
| Over-runs requirements phase, starves the deep dive | 23 | 2026-08-08 |
| Doesn't volunteer break/fix in deep dives | 18 | 2026-08-08 |
| Answers one part of a multi-part question per turn | 4 | 2026-08-08 |
| Cannot name a remaining bottleneck at wrap-up | 1 | 2026-08-08 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Mixed | 2026-08-08 |
| Leads with trade-offs vs alternatives | Mixed | 2026-08-08 |
| Pushes scale until it breaks | Mixed | 2026-08-08 |
| API as a designed contract | Mixed | 2026-08-08 |
| Operability / second-order concerns | Strong | 2026-08-08 |
| Pace (core by mid, deep dive after) | Weak | 2026-08-08 |
