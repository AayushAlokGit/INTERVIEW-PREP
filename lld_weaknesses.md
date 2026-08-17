# LLD Weaknesses
Last updated: 2026-08-17

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Requirements & Scoping
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Asks a walk question, then never answers it | 3 | 3 | 2026-08-17 |
| Never produces an Out of Scope list | 5 | 3 | 2026-08-13 |
| Doesn't state a concurrency posture | 3 | 2 | 2026-08-17 |
| Failure convention never named as a requirement | 2 | 2 | 2026-08-17 |
| Doesn't probe domain variants (screens, currency, size fit) | 3 | 2 | 2026-08-14 |

## Entity Modelling
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Models actors as classes that hold no rule | 4 | 4 | 2026-08-17 |
| No entity owns a rule he already stated | 3 | 3 | 2026-08-17 |
| Skips the entity a stated rule requires | 2 | 2 | 2026-08-17 |
| Abstraction placed at the smaller variation point | 1 | 1 | 2026-08-17 |

## Class Design & Encapsulation
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Incomplete signatures on first submission | 5 | 5 | 2026-08-17 |
| Declared contract contradicts method body | 3 | 3 | 2026-08-10 |
| No return types on mutators; entity nothing constructs | 2 | 2 | 2026-08-17 |
| Stated requirement has no state to enforce it | 2 | 2 | 2026-08-17 |
| Core operation has no entry point on the orchestrator | 1 | 2 | 2026-08-17 |

## Responsibility Placement
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Getters used so caller decides (Ask, not Tell) | 4 | 3 | 2026-08-17 |
| Owner object holds state it never acts on | 3 | 3 | 2026-08-17 |
| Same operation on two classes, no split stated | 2 | 2 | 2026-08-17 |

## Implementation & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| No self-trace before submitting | 3 | 3 | 2026-08-10 |
| Writes a method nothing ever calls | 1 | 1 | 2026-08-10 |
| Answers a trace request with a dependency, not values | 1 | 1 | 2026-08-10 |
| Trusts objects passed in by callers | 2 | 1 | 2026-08-08 |
| Defends a check that isn't in his own code | 1 | 1 | 2026-08-10 |

## Simplicity & Patterns
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| _(none observed — restraint remains a strength)_ | | | |

## Extensibility & Concurrency
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| States no cost for a chosen primitive | 3 | 3 | 2026-08-10 |
| Names the phase to lock, not the place | 1 | 0 | 2026-08-08 |

## Communication & Pace
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Composes prose so slowly the last item is cut | 3 | 3 | 2026-08-17 |
| Re-types entity fields already written once | 2 | 2 | 2026-08-17 |
| Serializes clarifying questions across turns | 2 | 2 | 2026-08-17 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Scopes before designing | Improving | 2026-08-17 |
| State derived from requirements | Mixed | 2026-08-17 |
| Rules live with their state (Tell, Don't Ask) | Improving | 2026-08-17 |
| Simplicity held under pressure | Strong | 2026-08-17 |
| Verifies own logic | Weak | 2026-08-10 |
| Extends without rewriting | Strong | 2026-08-10 |
