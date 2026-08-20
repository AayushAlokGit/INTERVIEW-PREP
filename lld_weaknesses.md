# LLD Weaknesses
Last updated: 2026-08-20

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Requirements & Scoping
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't state a concurrency posture | 6 | 5 | 2026-08-20 |
| Serializes the walk one question per turn | 2 | 2 | 2026-08-20 |
| Never asks the domain/value-variant question | 1 | 1 | 2026-08-20 |
| Failure convention never named as a requirement | 4 | 2 | 2026-08-20 |
| Never asks lifecycle / terminal states | 1 | 1 | 2026-08-20 |

## Entity Modelling
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Skips the entity a stated rule requires | 5 | 5 | 2026-08-20 |
| Entities arrive as a sketch, rebuilt under questioning | 1 | 1 | 2026-08-20 |
| Models actors as classes that hold no rule | 5 | 3 | 2026-08-19 |
| No entity owns a rule he already stated | 4 | 2 | 2026-08-19 |

## Class Design & Encapsulation
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Incomplete signatures on first submission | 8 | 8 | 2026-08-20 |
| Accessors added with no caller identified | 1 | 1 | 2026-08-20 |
| Entity has no state for its own lifecycle | 4 | 3 | 2026-08-19 |
| Declared contract contradicts method body | 4 | 2 | 2026-08-19 |
| Public API takes live objects, not ids | 1 | 0 | 2026-08-19 |

## Responsibility Placement
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Getters used so caller decides (Ask, not Tell) | 6 | 4 | 2026-08-19 |
| Owner object holds state it never acts on | 4 | 2 | 2026-08-19 |
| Same rule checked in two or three layers | 1 | 0 | 2026-08-19 |

## Implementation & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| No self-trace before submitting | 6 | 6 | 2026-08-20 |
| Cannot execute his own code on paper | 1 | 1 | 2026-08-20 |
| Trace contradicts his own stated contract | 1 | 1 | 2026-08-20 |
| Trusts objects passed in by callers | 4 | 2 | 2026-08-19 |
| Asserts a traced scenario works without running it | 2 | 2 | 2026-08-20 |

## Simplicity & Patterns
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| _(none observed — restraint remains a consistent strength)_ | | | |

## Extensibility & Concurrency
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Follow-up answered as a guard, not a type change | 1 | 1 | 2026-08-20 |
| Bolts a sub-expression property onto the owner | 1 | 1 | 2026-08-20 |
| Names the concurrency symptom, not the category | 2 | 1 | 2026-08-19 |
| Follow-ups answered as method names, not designs | 1 | 0 | 2026-08-19 |

## Communication & Pace
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Re-derives the same design two or three times | 1 | 1 | 2026-08-20 |
| Leaves interviewer questions unanswered across turns | 3 | 3 | 2026-08-20 |
| Re-types entity fields already written once | 4 | 3 | 2026-08-19 |
| Serializes clarifying questions across turns | 4 | 3 | 2026-08-19 |
| Composes prose so slowly the last item is cut | 4 | 2 | 2026-08-19 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Scopes before designing | Mixed | 2026-08-20 |
| State derived from requirements | Strong | 2026-08-20 |
| Rules live with their state (Tell, Don't Ask) | Strong | 2026-08-20 |
| Simplicity held under pressure | Strong | 2026-08-20 |
| Verifies own logic | Weak | 2026-08-20 |
| Extends without rewriting | Mixed | 2026-08-20 |
