# System Design Weaknesses
Last updated: 2026-08-17 (session 45 - design sprint: Instagram)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Arithmetic slips in BoE / SLA math | 24 | 2026-08-14 |
| Asserts a traffic model without sanity-checking it | 11 | 2026-08-17 |
| Omits read:write ratio, storage growth, durability | 6 | 2026-08-14 |
| Refuses to quantify when asked directly for a number | 6 | 2026-08-13 |
| Stops one step short of the number that decides it | 1 | 2026-08-17 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Response omits load-bearing fields (cursor, id, status) | 31 | 2026-08-17 |
| In-scope FR ships with reads but no write endpoint | 8 | 2026-08-17 |
| No error/status semantics on any endpoint | 6 | 2026-08-17 |
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
| Answers ~70% of a question, rest costs a round-trip | 8 | 2026-08-13 |
| Earlier phases overrun; the API phase pays the bill | 5 | 2026-08-17 |
| Writes the API in flow order, not from the FR list | 1 | 2026-08-17 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Weak | 2026-08-13 |
| Leads with trade-offs vs alternatives | Weak | 2026-08-13 |
| Pushes scale until it breaks | Weak | 2026-08-13 |
| API as a designed contract | Mixed — volunteered a presigned two-phase upload with a client-branchable status enum, cursor pagination with a stated justification, and an idempotency header, all unprompted; but an in-scope FR (likes/comments) shipped with reads and no writes, no delete or unfollow existed anywhere, no endpoint had error semantics, the one request body named an unstructured `photoMetadata`, and the feed returned `likeCount`/`commentCount` that no entity holds | 2026-08-17 |
| Operability / second-order concerns | Not observed | 2026-08-13 |
| Pace (core by mid, deep dive after) | Weak, improving — all three phases landed and entities came in under target for the first time (11:54 vs 12:00), but requirements took 54% of the box and the API overran to 20:06 (+3:06). First passes remain single-message with zero revision or back-fill, so pace is still a composition-throughput problem, not a packaging one; the API's missing categories (FR4 writes, deletes, errors) are a separate checklist gap, not a clock gap | 2026-08-17 |
