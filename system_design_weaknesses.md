# System Design Weaknesses
Last updated: 2026-08-17 (session 46 - design sprint: Online Auction)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Arithmetic slips in BoE / SLA math | 24 | 2026-08-14 |
| Asserts a traffic model without sanity-checking it | 12 | 2026-08-17 |
| Omits read:write ratio, storage growth, durability | 7 | 2026-08-17 |
| Asks for the givens, then derives nothing from them | 1 | 2026-08-17 |
| Stops one step short of the number that decides it | 2 | 2026-08-17 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Response omits load-bearing fields (cursor, id, status) | 32 | 2026-08-17 |
| In-scope FR ships with no endpoint or mechanism | 9 | 2026-08-17 |
| No error/status semantics on any endpoint | 7 | 2026-08-17 |
| No request shapes or named body fields on writes | 2 | 2026-08-17 |
| No idempotency on the most contended write | 4 | 2026-08-17 |

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
| Over-runs requirements phase, starves the deep dive | 33 | 2026-08-17 |
| Doesn't volunteer break/fix in deep dives | 18 | 2026-08-08 |
| Ends the phase early with walk items unwritten | 1 | 2026-08-17 |
| Earlier phases overrun; the API phase pays the bill | 5 | 2026-08-17 |
| Writes the API in flow order, not from the FR list | 2 | 2026-08-17 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Weak | 2026-08-13 |
| Leads with trade-offs vs alternatives | Weak | 2026-08-13 |
| Pushes scale until it breaks | Weak | 2026-08-13 |
| API as a designed contract | Mixed — idempotency landed on the single most contended write (the bid), cursor pagination was justified in-line, identity-from-auth stated once, and the created object returned on POST; but the bid endpoint has no response body at all, so no client can learn whether its bid was accepted; FR5's sub-second live-price NFR shipped with no delivery mechanism; `next_cursor` was justified and then not returned; and no endpoint has error semantics for bid-too-low, bid-after-close, or not-found | 2026-08-17 |
| Operability / second-order concerns | Not observed | 2026-08-13 |
| Pace (core by mid, deep dive after) | Improving — **first front half in the record to close inside 17:00** (15:00, voluntarily, two minutes unused), all three phases delivered, entities and API both under target. But the standing diagnosis inverts: this was fast-and-incomplete rather than slow-and-complete. Four of seven requirement walk items were omitted with time still on the clock, so the remaining problem is checklist coverage, not throughput — a different fix from the previous three sittings | 2026-08-17 |
