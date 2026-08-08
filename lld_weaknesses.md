# LLD Weaknesses
Last updated: 2026-08-08

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Requirements & Scoping
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't ask the error/contract semantics | 2 | 2 | 2026-08-08 |
| Asks interviewer what requirements he's missing | 1 | 1 | 2026-08-08 |
| Doesn't probe domain variants (currency, multi-payer) | 1 | 1 | 2026-08-08 |
| Omits given rules from written requirements | 1 | 1 | 2026-08-06 |

## Entity Modelling
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Entity list needs several revision passes | 1 | 1 | 2026-08-08 |

## Class Design & Encapsulation
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Incomplete signatures on first submission | 2 | 2 | 2026-08-08 |
| Declared contract contradicts method body | 2 | 2 | 2026-08-08 |
| Method references state its class doesn't hold | 1 | 1 | 2026-08-08 |
| Entities exposed as getter bags | 1 | 1 | 2026-08-08 |

## Responsibility Placement
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Getters used so caller decides (Ask, not Tell) | 2 | 2 | 2026-08-08 |
| Strategy mutates the input it was handed | 1 | 1 | 2026-08-08 |

## Implementation & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| No self-trace before submitting | 2 | 2 | 2026-08-08 |
| Trusts objects passed in by callers | 2 | 2 | 2026-08-08 |
| Defends an unworkable signature when challenged | 1 | 1 | 2026-08-08 |
| Float equality used as a control condition | 1 | 1 | 2026-08-08 |

## Simplicity & Patterns
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| _(none observed — restraint remains a strength)_ | | | |

## Extensibility & Concurrency
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| States no cost for a chosen primitive | 2 | 2 | 2026-08-08 |
| Names the phase to lock, not the place | 1 | 1 | 2026-08-08 |

## Communication & Pace
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Leaves a repeated probe unresolved | 1 | 1 | 2026-08-08 |
| Entities phase overruns via repeated revision | 1 | 1 | 2026-08-08 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Scopes before designing | Strong | 2026-08-08 |
| State derived from requirements | Mixed | 2026-08-08 |
| Rules live with their state (Tell, Don't Ask) | Mixed | 2026-08-08 |
| Simplicity held under pressure | Strong | 2026-08-08 |
| Verifies own logic | Mixed | 2026-08-08 |
| Extends without rewriting | Strong | 2026-08-08 |
