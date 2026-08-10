# LLD Weaknesses
Last updated: 2026-08-10

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Requirements & Scoping
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never produces an Out of Scope list | 3 | 3 | 2026-08-10 |
| Leaves a rule he raised without a resolution | 1 | 1 | 2026-08-10 |
| Doesn't ask whether the object is multi-threaded | 1 | 1 | 2026-08-10 |
| Doesn't probe domain variants (currency, multi-payer) | 1 | 1 | 2026-08-08 |

## Entity Modelling
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Entity list needs several revision passes | 2 | 2 | 2026-08-10 |
| Models actors as classes that hold no rule | 1 | 1 | 2026-08-10 |

## Class Design & Encapsulation
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Incomplete signatures on first submission | 3 | 3 | 2026-08-10 |
| Declared contract contradicts method body | 3 | 3 | 2026-08-10 |
| Method appears in the trace but not the class | 1 | 1 | 2026-08-10 |

## Responsibility Placement
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Getters used so caller decides (Ask, not Tell) | 3 | 3 | 2026-08-10 |
| Owner object holds state it never acts on | 1 | 1 | 2026-08-10 |

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
| Answers one part of a two-part question | 2 | 2 | 2026-08-10 |
| Entities phase overruns via repeated revision | 2 | 2 | 2026-08-10 |
| Leaves a repeated probe unresolved | 1 | 0 | 2026-08-08 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Scopes before designing | Mixed | 2026-08-10 |
| State derived from requirements | Strong | 2026-08-10 |
| Rules live with their state (Tell, Don't Ask) | Weak | 2026-08-10 |
| Simplicity held under pressure | Strong | 2026-08-10 |
| Verifies own logic | Weak | 2026-08-10 |
| Extends without rewriting | Strong | 2026-08-10 |
