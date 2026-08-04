# System Design Weaknesses
Last updated: 2026-08-04 (session 35 - Recommendation System)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Cannot derive scale numbers independently | 17 | 2026-07-28 |
| Arithmetic slips in BoE / SLA math | 16 | 2026-08-04 |
| Refuses to quantify when asked directly for a number | 1 | 2026-08-04 |
| Sizes only one read surface, misses the larger one | 1 | 2026-08-04 |
| Never states what is out of scope | 1 | 2026-08-04 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Vague response shape (no explicit fields) | 21 | 2026-08-04 |
| Missing pagination on list endpoints | 6 | 2026-08-04 |
| Names an endpoint without specifying its shape | 2 | 2026-07-29 |
| Omits whole resource from the API (e.g. invoices) | 2 | 2026-07-29 |
| Client-supplied identity accepted from request body | 1 | 2026-08-04 |

## Deep Dives
| Weakness | Sessions | Last Seen |
|---|---|---|
| Asks for hints / leans on interviewer | 18 | 2026-08-04 |
| Hand-waves core algorithm (geo, sharding, consensus) | 11 | 2026-08-04 |
| Dismisses a second FR as "similar flow", never designs it | 1 | 2026-08-04 |
| Doesn't size the dependency or cost he introduces | 3 | 2026-08-04 |
| Stops at first fix; no trade-offs on the new design | 2 | 2026-07-29 |

## Architecture & Trade-offs
| Weakness | Sessions | Last Seen |
|---|---|---|
| States choices without naming the alternative | 18 | 2026-08-04 |
| Missing resilience patterns (DLQ, breaker, failover) | 12 | 2026-08-04 |
| Picks store contradicting access pattern (SQL for KV lookup) | 5 | 2026-08-04 |
| Uses a database as a stream source (scatter-gather scan) | 1 | 2026-08-04 |
| Doesn't self-catch hot-key/partition in own design | 3 | 2026-07-29 |

## Communication & Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Requires multiple prompts to fully articulate ideas | 24 | 2026-08-04 |
| Over-runs time budget (60-70 min vs 45-50) | 21 | 2026-08-04 |
| Doesn't volunteer break/fix in deep dives | 16 | 2026-08-04 |
| Leaves the last question in a list unanswered | 3 | 2026-08-04 |
| Long silence (20+ min) on a single turn | 1 | 2026-08-04 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Mixed | 2026-08-04 |
| Leads with trade-offs vs alternatives | Weak | 2026-08-04 |
| Pushes scale until it breaks | Weak | 2026-08-04 |
| API as a designed contract | Mixed | 2026-08-04 |
| Operability / second-order concerns | Mixed | 2026-08-04 |
| Pace (core by mid, deep dive after) | Weak | 2026-08-04 |
