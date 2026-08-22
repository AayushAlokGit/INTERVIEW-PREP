# LLD Weaknesses
Last updated: 2026-08-22

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Requirements & Scoping
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't state a concurrency posture | 7 | 6 | 2026-08-22 |
| Failure convention never named as a requirement | 5 | 3 | 2026-08-22 |
| Serializes the walk one question per turn | 3 | 3 | 2026-08-22 |
| Never asks the domain/value-variant question | 2 | 2 | 2026-08-22 |
| Never asks lifecycle / terminal states | 2 | 2 | 2026-08-22 |

## Entity Modelling
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Skips the entity a stated rule requires | 5 | 4 | 2026-08-20 |
| Models actors as classes that hold no rule | 6 | 4 | 2026-08-22 |
| Entities arrive as a sketch, rebuilt under questioning | 2 | 2 | 2026-08-22 |
| No entity owns a rule he already stated | 4 | 1 | 2026-08-19 |

## Class Design & Encapsulation
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Incomplete signatures on first submission | 9 | 9 | 2026-08-22 |
| Entity has no state for its own lifecycle | 5 | 4 | 2026-08-22 |
| Declared contract contradicts method body | 5 | 3 | 2026-08-22 |
| Interface declares a concrete implementation's field | 1 | 1 | 2026-08-22 |

## Responsibility Placement
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Getters used so caller decides (Ask, not Tell) | 6 | 3 | 2026-08-19 |
| Mutating method named as a plain accessor | 1 | 1 | 2026-08-22 |
| Owner object holds state it never acts on | 4 | 1 | 2026-08-19 |

## Implementation & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| No self-trace before submitting | 7 | 7 | 2026-08-22 |
| Asserts a traced scenario works without running it | 3 | 3 | 2026-08-22 |
| Cannot execute his own code on paper | 2 | 2 | 2026-08-22 |
| Trace contradicts his own stated contract | 2 | 2 | 2026-08-22 |
| Trusts objects passed in by callers | 4 | 1 | 2026-08-19 |

## Simplicity & Patterns
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Adds an abstraction he admits no requirement needs | 1 | 1 | 2026-08-22 |

## Extensibility & Concurrency
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| _(none active — all rows retired 2026-08-22; concurrency answered at the senior bar)_ | | | |

## Communication & Pace
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Leaves interviewer questions unanswered across turns | 4 | 4 | 2026-08-22 |
| Serializes clarifying questions across turns | 5 | 4 | 2026-08-22 |
| Composes prose so slowly the last item is cut | 5 | 3 | 2026-08-22 |
| Re-types entity fields already written once | 4 | 2 | 2026-08-19 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Scopes before designing | Mixed | 2026-08-22 |
| State derived from requirements | Mixed | 2026-08-22 |
| Rules live with their state (Tell, Don't Ask) | Strong | 2026-08-22 |
| Simplicity held under pressure | Weak | 2026-08-22 |
| Verifies own logic | Weak | 2026-08-22 |
| Extends without rewriting | Strong | 2026-08-22 |
