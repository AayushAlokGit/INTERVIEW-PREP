# System Design Weaknesses
Last updated: 2026-09-30 (session 51 — system design round: Online Chess)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Arithmetic slips in BoE / SLA math | 28 | 2026-09-30 |
| Asserts a traffic model without sanity-checking it | 16 | 2026-09-30 |
| Stops one step short of the number that decides it | 5 | 2026-09-30 |
| Produces zero NFRs / no numbers at all | 1 | 2026-08-22 |
| Never asks for scale givens when invited | 1 | 2026-08-22 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Response omits load-bearing fields (cursor, id, status) | 34 | 2026-09-30 |
| In-scope FR ships with no endpoint or mechanism | 11 | 2026-09-30 |
| No error/status semantics on any endpoint | 10 | 2026-09-30 |
| No idempotency key on a retryable write | 4 | 2026-09-30 |
| No position on cursor stability under concurrent writes | 1 | 2026-08-20 |

## Deep Dives
| Weakness | Sessions | Last Seen |
|---|---|---|
| Asks for hints / leans on interviewer | 23 | 2026-09-29 |
| Hand-waves core algorithm (geo, sharding, consensus) | 14 | 2026-09-30 |
| Declines to attempt sizing before being pushed | 8 | 2026-09-30 |
| Moves on from a break he just identified | 2 | 2026-09-29 |
| Names a mitigation instead of resolving the break | 3 | 2026-09-30 |

## Architecture & Trade-offs
| Weakness | Sessions | Last Seen |
|---|---|---|
| States choices without naming the alternative | 27 | 2026-09-30 |
| Missing resilience patterns (DLQ, breaker, failover) | 17 | 2026-09-30 |
| Names a datastore category, never a concrete system | 3 | 2026-08-19 |
| Trade-offs on own inventions, none on dependencies | 1 | 2026-08-20 |
| Makes a scoping decision silently instead of stating it | 1 | 2026-08-20 |

## Communication & Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Over-runs requirements phase, starves the deep dive | 37 | 2026-09-30 |
| Doesn't volunteer break/fix in deep dives | 20 | 2026-09-30 |
| FRs restate the prompt instead of making choices | 3 | 2026-09-30 |
| Never states what is out of scope | 3 | 2026-09-30 |
| Abandons the round mid-phase instead of pushing through | 2 | 2026-09-30 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Mixed — self-raised Idempotency-Key on matching, GameMatchingRequest entity, pub/sub at-most-once, and clocks as the hard part; but HLD shipped with no inbound move path/Postgres writer/notification channel, and 'unsure' 4× in the deep dive | 2026-09-30 |
| Leads with trade-offs vs alternatives | Weak — only pub/sub vs polling argued; no alternative named for in-memory state, consistent hashing, Kafka, Postgres, Redis queues | 2026-09-30 |
| Pushes scale until it breaks | Weak — games/s double-counted players (500 vs 250), no moves/s, peak or connection count; 1M-spectator fan-out left unsized | 2026-09-30 |
| API as a designed contract | Mixed — Idempotency-Key + async id/status + WS command vocabulary; sendMove has no moveNumber, no clock/game-over messages, no top-live-games endpoint | 2026-09-30 |
| Operability / second-order concerns | Weak — no failover detection, fencing, reconnect, consumer lag, or any metric raised | 2026-09-30 |
| Pace (core by mid, deep dive after) | Weak — API at 33, HLD at 55 vs 27, round 70 min; ~33 min of round-trip tax | 2026-09-30 |
