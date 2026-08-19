# LLD Weaknesses
Last updated: 2026-08-19

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Requirements & Scoping
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't state a concurrency posture | 5 | 4 | 2026-08-19 |
| Asks the interviewer for operations and rules | 1 | 1 | 2026-08-19 |
| Never produces an Out of Scope list | 5 | 1 | 2026-08-13 |
| Failure convention never named as a requirement | 3 | 1 | 2026-08-19 |

## Entity Modelling
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Models actors as classes that hold no rule | 5 | 4 | 2026-08-19 |
| No entity owns a rule he already stated | 4 | 3 | 2026-08-19 |
| Skips the entity a stated rule requires | 4 | 4 | 2026-08-19 |

## Class Design & Encapsulation
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Incomplete signatures on first submission | 7 | 7 | 2026-08-19 |
| Declared contract contradicts method body | 4 | 3 | 2026-08-19 |
| Entity has no state for its own lifecycle | 4 | 4 | 2026-08-19 |
| Public API takes live objects, not ids | 1 | 1 | 2026-08-19 |
| Core operation has no entry point on the orchestrator | 1 | 0 | 2026-08-17 |

## Responsibility Placement
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Same rule checked in two or three layers | 1 | 1 | 2026-08-19 |
| Getters used so caller decides (Ask, not Tell) | 6 | 5 | 2026-08-19 |
| Owner object holds state it never acts on | 4 | 3 | 2026-08-19 |

## Implementation & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| No self-trace before submitting | 5 | 5 | 2026-08-19 |
| Trusts objects passed in by callers | 4 | 3 | 2026-08-19 |
| Second loop unchecked against first loop's mutations | 1 | 1 | 2026-08-19 |
| Asserts a traced scenario works without running it | 1 | 1 | 2026-08-19 |

## Simplicity & Patterns
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| _(none observed — restraint remains a consistent strength)_ | | | |

## Extensibility & Concurrency
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Names the concurrency symptom, not the category | 2 | 2 | 2026-08-19 |
| Follow-ups answered as method names, not designs | 1 | 1 | 2026-08-19 |
| States no cost for a chosen primitive | 3 | 1 | 2026-08-10 |

## Communication & Pace
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Re-types entity fields already written once | 4 | 4 | 2026-08-19 |
| Serializes clarifying questions across turns | 4 | 4 | 2026-08-19 |
| Composes prose so slowly the last item is cut | 4 | 3 | 2026-08-19 |
| Leaves interviewer questions unanswered across turns | 2 | 2 | 2026-08-19 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Scopes before designing | Mixed | 2026-08-19 |
| State derived from requirements | Strong | 2026-08-19 |
| Rules live with their state (Tell, Don't Ask) | Mixed | 2026-08-19 |
| Simplicity held under pressure | Strong | 2026-08-19 |
| Verifies own logic | Weak | 2026-08-19 |
| Extends without rewriting | Strong | 2026-08-19 |
