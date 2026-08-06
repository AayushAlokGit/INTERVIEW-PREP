# System Design Weaknesses
Last updated: 2026-08-06 (session 36 - Logging & Telemetry Pipeline)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Cannot derive scale numbers independently | 18 | 2026-08-06 |
| Arithmetic slips in BoE / SLA math | 17 | 2026-08-06 |
| Refuses to quantify when asked directly for a number | 2 | 2026-08-06 |
| Never states what is out of scope | 2 | 2026-08-06 |
| Doesn't reconcile NFRs that contradict each other | 1 | 2026-08-06 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Never produces an API at all when time-pressured | 1 | 2026-08-06 |
| Vague response shape (no explicit fields) | 21 | 2026-08-04 |
| Missing pagination on list endpoints | 6 | 2026-08-04 |
| Names an endpoint without specifying its shape | 2 | 2026-07-29 |
| Client-supplied identity accepted from request body | 1 | 2026-08-04 |

## Deep Dives
| Weakness | Sessions | Last Seen |
|---|---|---|
| Asks for hints / leans on interviewer | 19 | 2026-08-06 |
| Abandons the round rather than attempting an answer | 1 | 2026-08-06 |
| Hand-waves core algorithm (geo, sharding, consensus) | 11 | 2026-08-04 |
| Doesn't size the dependency or cost he introduces | 3 | 2026-08-04 |
| Stops at first fix; no trade-offs on the new design | 2 | 2026-07-29 |

## Architecture & Trade-offs
| Weakness | Sessions | Last Seen |
|---|---|---|
| States choices without naming the alternative | 19 | 2026-08-06 |
| Names a datastore category, never a concrete system | 1 | 2026-08-06 |
| Missing resilience patterns (DLQ, breaker, failover) | 13 | 2026-08-06 |
| Designs only one FR's path, leaves the rest undrawn | 1 | 2026-08-06 |
| Doesn't self-catch hot-key/partition in own design | 4 | 2026-08-06 |

## Communication & Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Requires multiple prompts to fully articulate ideas | 25 | 2026-08-06 |
| Over-runs requirements phase, starves the deep dive | 22 | 2026-08-06 |
| Doesn't volunteer break/fix in deep dives | 17 | 2026-08-06 |
| Defends a wrong number instead of re-checking it | 1 | 2026-08-06 |
| Leaves the last question in a list unanswered | 3 | 2026-08-04 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Weak | 2026-08-06 |
| Leads with trade-offs vs alternatives | Weak | 2026-08-06 |
| Pushes scale until it breaks | Weak | 2026-08-06 |
| API as a designed contract | Weak | 2026-08-06 |
| Operability / second-order concerns | Weak | 2026-08-06 |
| Pace (core by mid, deep dive after) | Weak | 2026-08-06 |
