# LLD Weaknesses
Last updated: 2026-08-19

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Requirements & Scoping
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't state a concurrency posture | 4 | 3 | 2026-08-19 |
| Failure convention never named as a requirement | 3 | 3 | 2026-08-19 |
| Asks a walk question, then never answers it | 3 | 2 | 2026-08-17 |
| Never produces an Out of Scope list | 5 | 2 | 2026-08-13 |
| Doesn't probe domain variants (screens, currency, size fit) | 3 | 1 | 2026-08-14 |

## Entity Modelling
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Models actors as classes that hold no rule | 5 | 5 | 2026-08-19 |
| No entity owns a rule he already stated | 4 | 4 | 2026-08-19 |
| Skips the entity a stated rule requires | 3 | 3 | 2026-08-19 |

## Class Design & Encapsulation
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Incomplete signatures on first submission | 6 | 6 | 2026-08-19 |
| Declared contract contradicts method body | 4 | 4 | 2026-08-19 |
| No return types on mutators; entity nothing constructs | 3 | 3 | 2026-08-19 |
| Stated requirement has no state to enforce it | 3 | 3 | 2026-08-19 |
| Core operation has no entry point on the orchestrator | 1 | 1 | 2026-08-17 |

## Responsibility Placement
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Getters used so caller decides (Ask, not Tell) | 5 | 4 | 2026-08-19 |
| Owner object holds state it never acts on | 4 | 4 | 2026-08-19 |
| Same operation on two classes, no split stated | 2 | 1 | 2026-08-17 |

## Implementation & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| No self-trace before submitting | 4 | 4 | 2026-08-19 |
| Trusts objects passed in by callers | 3 | 2 | 2026-08-19 |
| Two distinct failures collapsed into one exception | 1 | 1 | 2026-08-19 |

## Simplicity & Patterns
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| _(none observed — restraint remains a strength)_ | | | |

## Extensibility & Concurrency
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| States no cost for a chosen primitive | 3 | 2 | 2026-08-10 |
| Names the concurrency symptom, not the category | 1 | 1 | 2026-08-19 |

## Communication & Pace
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Composes prose so slowly the last item is cut | 4 | 4 | 2026-08-19 |
| Re-types entity fields already written once | 3 | 3 | 2026-08-19 |
| Serializes clarifying questions across turns | 3 | 3 | 2026-08-19 |
| Leaves interviewer questions unanswered across turns | 1 | 1 | 2026-08-19 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Scopes before designing | Strong | 2026-08-19 |
| State derived from requirements | Mixed | 2026-08-19 |
| Rules live with their state (Tell, Don't Ask) | Mixed | 2026-08-19 |
| Simplicity held under pressure | Strong | 2026-08-19 |
| Verifies own logic | Mixed | 2026-08-19 |
| Extends without rewriting | Strong | 2026-08-19 |
