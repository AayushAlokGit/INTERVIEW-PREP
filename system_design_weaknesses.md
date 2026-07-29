# System Design Weaknesses
Last updated: 2026-07-29 (session 34 - API Gateway)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Cannot derive scale numbers independently | 17 | 2026-07-28 |
| Arithmetic slips in BoE / SLA math | 15 | 2026-07-29 |
| NFRs stated too late without prompting | 13 | 2026-07-28 |
| No latency budget for what the system itself adds | 1 | 2026-07-29 |
| Ignores stated skew/peak in later design | 1 | 2026-07-28 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Vague response shape (no explicit fields) | 20 | 2026-07-29 |
| Names an endpoint without specifying its shape | 2 | 2026-07-29 |
| Omits whole resource from the API (e.g. invoices) | 2 | 2026-07-29 |
| No authorisation on admin / control-plane endpoints | 1 | 2026-07-29 |
| Missing pagination on list endpoints | 5 | 2026-07-28 |

## Deep Dives
| Weakness | Sessions | Last Seen |
|---|---|---|
| Asks for hints / leans on interviewer | 17 | 2026-07-29 |
| Hand-waves core algorithm (geo, sharding, consensus) | 10 | 2026-07-29 |
| Doesn't size the dependency or cost he introduces | 2 | 2026-07-29 |
| Stops at first fix; no trade-offs on the new design | 2 | 2026-07-29 |
| Durability ordering: sync-vs-async confusion | 1 | 2026-06-15 |

## Architecture & Trade-offs
| Weakness | Sessions | Last Seen |
|---|---|---|
| States choices without naming the alternative | 17 | 2026-07-29 |
| Violates his own stated principle elsewhere in design | 1 | 2026-07-29 |
| Missing resilience patterns (DLQ, breaker, failover) | 11 | 2026-07-28 |
| Doesn't self-catch hot-key/partition in own design | 3 | 2026-07-29 |
| Picks store contradicting access pattern (mutable sort key) | 4 | 2026-07-29 |

## Communication & Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Requires multiple prompts to fully articulate ideas | 23 | 2026-07-29 |
| Over-runs time budget (60-70 min vs 45-50) | 20 | 2026-07-29 |
| Doesn't volunteer break/fix in deep dives | 15 | 2026-07-29 |
| Leaves the last question in a list unanswered | 2 | 2026-07-29 |
| Requirements phase eats the deep-dive budget | 2 | 2026-07-29 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Weak | 2026-07-29 |
| Leads with trade-offs vs alternatives | Mixed | 2026-07-29 |
| Pushes scale until it breaks | Weak | 2026-07-29 |
| API as a designed contract | Weak | 2026-07-29 |
| Operability / second-order concerns | Weak | 2026-07-29 |
| Pace (core by mid, deep dive after) | Weak | 2026-07-29 |
