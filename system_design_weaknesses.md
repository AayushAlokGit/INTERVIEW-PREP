# System Design Weaknesses
Last updated: 2026-07-28 (session 33 - Usage-Based Metering & Billing System)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Cannot derive scale numbers independently | 17 | 2026-07-28 |
| Arithmetic slips in BoE / SLA math | 14 | 2026-07-28 |
| NFRs stated too late without prompting | 13 | 2026-07-28 |
| Ignores stated skew/peak in later design | 1 | 2026-07-28 |
| Doesn't restate numbers after correcting them | 1 | 2026-07-28 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Vague response shape (no explicit fields) | 19 | 2026-07-28 |
| Missing pagination on list endpoints | 5 | 2026-07-28 |
| Omits whole resource from the API (e.g. invoices) | 1 | 2026-07-28 |
| Names an endpoint without specifying its shape | 1 | 2026-07-28 |
| HTTP verb mistakes (PUT vs PATCH/DELETE) | 5 | 2026-06-10 |

## Deep Dives
| Weakness | Sessions | Last Seen |
|---|---|---|
| Asks for hints / leans on interviewer | 16 | 2026-06-22 |
| Hand-waves core algorithm (geo, sharding, consensus) | 9 | 2026-06-11 |
| Stops at first fix; no trade-offs on the new design | 1 | 2026-07-28 |
| Doesn't size the read cost he shifts to read time | 1 | 2026-07-28 |
| Durability ordering: sync-vs-async confusion | 1 | 2026-06-15 |

## Architecture & Trade-offs
| Weakness | Sessions | Last Seen |
|---|---|---|
| States choices without naming the alternative | 16 | 2026-07-28 |
| Missing resilience patterns (DLQ, breaker, failover) | 11 | 2026-07-28 |
| Doesn't self-catch hot-key/partition in own design | 2 | 2026-07-28 |
| No partitioning/sharding scheme proposed at all | 1 | 2026-07-28 |
| Picks store contradicting access pattern (mutable sort key) | 3 | 2026-06-19 |

## Communication & Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Requires multiple prompts to fully articulate ideas | 22 | 2026-07-28 |
| Over-runs time budget (60-70 min vs 45-50) | 19 | 2026-06-19 |
| Doesn't volunteer break/fix in deep dives | 14 | 2026-06-15 |
| Answers one of two questions asked | 1 | 2026-07-28 |
| Requirements phase eats the deep-dive budget | 1 | 2026-07-28 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Mixed | 2026-07-28 |
| Leads with trade-offs vs alternatives | Weak | 2026-07-28 |
| Pushes scale until it breaks | Weak | 2026-07-28 |
| API as a designed contract | Weak | 2026-07-28 |
| Operability / second-order concerns | Weak | 2026-07-28 |
| Pace (core by mid, deep dive after) | Weak | 2026-07-28 |
