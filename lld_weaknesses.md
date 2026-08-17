# LLD Weaknesses
Last updated: 2026-08-17

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Requirements & Scoping
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never produces an Out of Scope list | 5 | 4 | 2026-08-13 |
| Leaves a rule he raised without a resolution | 2 | 1 | 2026-08-13 |
| Doesn't probe domain variants (screens, multi-seat, size fit) | 3 | 3 | 2026-08-17 |
| Failure convention never named as a requirement | 1 | 1 | 2026-08-17 |
| Lists one actor without probing for a second | 1 | 1 | 2026-08-17 |

## Entity Modelling
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Models actors as classes that hold no rule | 3 | 3 | 2026-08-17 |
| No entity owns a rule he already stated | 2 | 2 | 2026-08-17 |
| Skips the intermediate container a rule needs | 1 | 1 | 2026-08-17 |

## Class Design & Encapsulation
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Incomplete signatures on first submission | 4 | 4 | 2026-08-17 |
| Declared contract contradicts method body | 3 | 3 | 2026-08-10 |
| Method appears in the trace but not the class | 1 | 1 | 2026-08-10 |
| No return types on mutators; entity nothing constructs | 1 | 1 | 2026-08-17 |
| Stated requirement has no state to enforce it | 1 | 1 | 2026-08-17 |

## Responsibility Placement
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Getters used so caller decides (Ask, not Tell) | 4 | 4 | 2026-08-17 |
| Owner object holds state it never acts on | 2 | 2 | 2026-08-17 |
| Same operation on two classes, no split stated | 1 | 1 | 2026-08-17 |

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
| Composes prose so slowly the last item is cut | 2 | 2 | 2026-08-17 |
| Re-types entity fields already written once | 1 | 1 | 2026-08-17 |
| Serializes clarifying questions across turns | 1 | 1 | 2026-08-17 |
| Answers one part of a two-part question | 2 | 1 | 2026-08-10 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Scopes before designing | Improving | 2026-08-17 |
| State derived from requirements | Mixed | 2026-08-17 |
| Rules live with their state (Tell, Don't Ask) | Weak | 2026-08-17 |
| Simplicity held under pressure | Strong | 2026-08-17 |
| Verifies own logic | Weak | 2026-08-10 |
| Extends without rewriting | Strong | 2026-08-10 |
