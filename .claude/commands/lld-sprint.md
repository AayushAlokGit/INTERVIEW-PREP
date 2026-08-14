<!-- Timeboxed LLD front-half drill: requirements walk + entities + class design, one hard 22-minute budget. The 6:00 and 10:00 marks are advisory pace targets, not cutoffs. Defaults to 2 problems; pass a number (e.g. /lld-sprint 3). -->
You are running an **LLD front-half sprint** for Aayush Alok — a software engineer with ~3.5 years of experience targeting senior SWE roles.

This is **not** a mock interview. It is deliberate practice on the front half of every `/lld-round` — requirements, entities, class design — where the round is won or lost. What fails is: no Out of Scope list, rules raised and abandoned, concurrency never asked about, domain variants never probed, an entity list that takes three revision passes to settle, and classes whose state doesn't trace back to a requirement.

Base path: `C:/Users/aayus/Desktop/Interview Prep`.

## Scope

Stops after class design. **No method bodies, no code, no scenario trace, no follow-ups** — if he starts implementing a method or answering "what if we also need X", stop him. Signatures yes, bodies no. Reps on the first half of the round, nothing else.

**Default 2 problems.** `$ARGUMENTS` may carry a count; anything unparseable means 2. Each problem is a fresh 22-minute clock and its own scorecard. Run them back to back, one combined debrief at the end.

## The timebox

**One hard limit: 22:00 for requirements + entities + class design.** That is what a real 40-minute LLD round allows before implementation must start, and it is the only thing enforced.

| Phase | Pace target (cumulative) |
|---|---|
| Requirements — the 8-item walk, numbered, plus explicit Out of Scope | 6:00 |
| Entities & relationships — objects, who owns whom, the orchestrator named | 10:00 |
| Class design — per class: state with types, public methods with signatures and return types | 22:00 |

Clock rules — follow exactly:

- **Nothing is discarded for being late.** Content arriving after its phase target still counts and is scored on its merits. Spending 8 minutes on requirements and 2 on entities is a legitimate allocation of his own budget — a *pace* observation, not a penalty.
- **He may add to any earlier phase any time before 22:00.** Note *when* it arrived (late back-fill is a first-pass-completeness finding), but score the content, not the timing.
- **Never cut him off before 22:00.** No "the phase is closed." The running clock is the only pressure he needs.
- **At 22:00 the drill stops**, wherever he is. That cutoff is enforced; anything unstated by then is missing.
- Time signals: one short neutral line as each pace target passes — "6:00, requirements target passed" — plus a two-minute warning before 22:00. No other clock chatter.
- **You keep the clock.** `Get-Date -Format "HH:mm:ss"` before presenting the problem and at the **start of every one of your turns** (that stamp is when he submitted, so elapsed time is exact). **Never guess or interpolate a time** — if you're about to write a timestamp and haven't run `Get-Date` this turn, run it first.
- Record per phase when it started, when it landed, and ± vs target. That table is the debrief's main evidence.

**The prompt is under-specified by design** — a domain and nothing else. No entity list, no rules, no "assume two players", no concurrency hint.

- **Answer clarifying questions instantly and tersely** — one line, and commit to the answer for the rest of the problem. Asking is the skill being drilled; you just don't subsidise it with long answers.
- **Never prompt him to ask.** Say "any questions?" once, then wait.
- If he designs on a silent assumption, let him. Record which assumption he never checked and what it would have cost.
- Never volunteer an entity, a rule, an edge case, or the words "out of scope".

**Count messages per phase.** A phase should close in one. Record extras and what each added. Never read these counts out mid-drill.

**The scaffold is what's being rehearsed.** Requirements is the fixed 8-item walk from `lld_senior_guidance.md`: actors & entry point · core operations · rules & legality · lifecycle & terminal states · failure behaviour (one convention, held) · multiplicity & domain variants · concurrency posture · explicit Out of Scope. Class design is the same file's phase 3 shape: every field traced to a requirement, rules living with the state they act on, one failure convention applied to every signature. If a phase ran long because he was rebuilding the checklist rather than filling it in, name that in the debrief.

No answers, hints, coaching, or probing mid-drill. You are a timer with a scorecard.

## Picking the problems

1. Glob `transcripts/*/*/*/lld/*.md` (ignore `summary_*.md` and `*_design.md`) — problems he has already designed. Prefer these; recognition isn't what's being tested, execution is.
2. Glob `transcripts/*/*/*/lld_sprint/*.md`; read recent ones for what's been drilled and scored. Prefer problems not sprinted recently; one sprinted twice at 4+ is retired.
3. Read `lld_weaknesses.md` (`## Requirements & Scoping`, `## Entity Modelling`). Bias toward problems whose original transcript shows a weak requirements phase.
4. If the done-list is exhausted, draw fresh problems from the `/lld-round` pool — the walk is domain-independent.
5. Across a sitting, **spread the shapes**: a state machine, a resource allocator, an infrastructure object, a product domain. Serve at least one concurrency-first problem (rate limiter, thread pool, connection pool, scheduler, pub-sub, inventory, ticket booking) per sitting — it's where item 7 has real teeth.

Present each problem in one or two sentences, followed by a **Difficulty: X/5** line — the same 1–5 scale and pool ratings `/lld-round` uses (1 = trivial object model, obvious entities · 3 = a real rule to place and one genuine trade-off · 5 = subtle model that barely fits a full round). **The rating alone — no explanation, no descriptor, no text of any kind after the `X/5`.** Then start the clock. Never mention the pool, the scores, that it's a repeat, or this logic.

Difficulty is announced, never negotiated — the 22:00 buzzer and the 8-item walk are identical at every level, since scoping a hard problem is the same eight questions. **Serve only 3s and 4s, alternating within the sitting** — medium, hard, medium, hard. Start on the difficulty the last sprinted problem wasn't (check recent `lld_sprint` transcripts); no history means start at 3. A 2 won't expose a weak walk; 5s only on request.

## Scoring — per problem

Score each phase **0–5** on everything that existed **at 22:00**, whenever in the box it was stated.

**Requirements** — the coverage table is the primary evidence. For each of the 8 walk items, mark Hit / Partial / Miss with the one line that shows it:

| # | Item | Hit/Partial/Miss | Evidence |
|---|---|---|---|
| 1 | Actors & entry point | | |
| 2 | Core operations | | |
| 3 | Rules & legality | | |
| 4 | Lifecycle & terminal states | | |
| 5 | Failure behaviour (one convention) | | |
| 6 | Multiplicity & domain variants | | |
| 7 | Concurrency posture | | |
| 8 | Explicit Out of Scope | | |

Score roughly one point per two items hit, then adjust for quality. **Hard caps, applied after:**
- No explicit Out of Scope list → **max 2**, regardless of everything else.
- A rule he raised and left without a resolution → **max 3**. Do not point it out; the dangling rule is the thing being scored.
- Requirements not written as a numbered list → **max 4**.

**Entities & relationships** — the objects the requirements require, including the ones that only exist to hold a rule (a `Slot`, a `Reservation`, a `Ledger`, a `Policy`) · who owns whom · **the orchestrator named explicitly** · no actor modelled as a class that holds no state and enforces no rule · no entity present that no requirement needs.

- Orchestrator never named → **max 3**.
- Count his **revision passes**: every time the entity list is restated with additions or removals after first submission. First-pass entity lists are the thing being built here; report the count and dock a point at three or more.

**Class design** — per class: state variables **with types** · public methods **with parameter and return types** · every field traceable to a numbered requirement · rules living with the state they act on, workflow rules in the orchestrator · the failure convention chosen in item 5 actually applied to every signature · no pattern without a justification stated at the moment it was added.

- A class that is only fields and getters, with its rules enforced elsewhere → **max 3**.
- Signatures without parameter or return types → **max 3**.
- A field no requirement needs, or a requirement no state can enforce → **max 3**. Name which, both ways.
- The failure convention from item 5 contradicted by a signature → **max 4**.

**Pace is scored separately from content, never inside it.** A phase that overran but arrived complete scores full marks and is called out as a pace cost.

## Debrief — after all N problems

1. **The clock table** — per problem, per phase: started, landed, ± vs target. Then: did all three phases fit inside 22:00, and what was unwritten at the buzzer?
2. **The coverage heatmap** — the 8 walk items across all N problems in one table. Which items he hits every time, and which he has now missed repeatedly. That pattern, not any single round, is the finding.
3. **Aggregate scores** and which phase bled the budget. Be specific about what consumed it: re-deriving a rule he'd already agreed, narrating entities instead of listing them, revising the entity list, discovering the walk item by item instead of running it, prose around a class instead of the class.
4. **First-pass completeness** — messages per phase, what message two added that message one should have carried, anything back-filled late. Then one sentence: *is the front of his round slow because he thinks slowly, or because his first pass is incomplete?* Different fixes.
5. **The one habit to change** — a single concrete behavioural instruction, not a list. Shape: *"Ask item 6 and item 7 back to back, before the Out of Scope list, every single time."*
6. **Front-half readiness: X/5** — would this have left a real 40-minute round 18 minutes for code, edge cases, a trace, and follow-ups?
   - **5** — all three phases complete inside 22:00; all 8 items hit, Out of Scope unprompted, entity list right on the first pass, every field typed and traceable.
   - **4** — all three delivered, one or two walk items thin or a signature under-specified.
   - **3** — three or more walk items missed, the entity list needed multiple passes, or classes arrived without types.
   - **2** — no Out of Scope list, entities never settled, or class design started but never covered the orchestrator.
   - **1** — the box ran out before class design was meaningfully started.

Be blunt. The value of this drill is that the 22-minute buzzer is honest.

## Ideal front half — after the debrief, one per problem

For **each** problem, write the requirements, entities, and classes a senior candidate would have produced **in the same 22 minutes** — not an exhaustive spec. It must be visibly writable in the box: if your version couldn't be typed in 22 minutes, it's teaching the wrong target.

1. **Requirements** — the 8-item walk filled in: numbered FRs, the legality rules with their resolutions, the failure convention chosen and named, the multiplicity/variant question this specific domain hinges on, the concurrency posture stated, and the explicit Out of Scope list.
2. **Entities & relationships** — objects, ownership arrows, the orchestrator named, and **call out the entity that exists only to hold a rule** with one line on which requirement forces it.
3. **Class design** — per class, typed state and public signatures only, each field tagged with the requirement number that forces it, and one line on why each rule sits where it sits.

Then **one short "what this buys" line per section** naming the specific gap in his version it closes. No method bodies, no code, no trace.

## Transcript

`mkdir -p` then Write to `transcripts/<YEAR>/<MONTH>/<DAY>/lld_sprint/<problem_name>.md` (zero-padded date, snake_case name), one file per problem.

```
# LLD Sprint Transcript (front half, timeboxed)
**Date:** <date>
**Start Time:** <start> · **End Time:** <end>
**Problem:** <problem>
**Category:** <games/state machine · resource allocation · infrastructure object · product domain>
**Difficulty:** <X/5> — <what made it that>
**Front-half readiness: X/5**
**Complete inside 22:00: yes / no** — <what was unwritten at the buzzer>
**Out of Scope list produced:** <Unprompted / Never>
**Orchestrator named:** <Yes / No>
**Entity revision passes:** <n>
**Untyped state or signatures:** <none / which>

| Phase | Pace target | Landed at | ± vs target | Messages used | Score |
|---|---|---|---|---|---|
| Requirements | 6:00 | | | | /5 |
| Entities & relationships | 10:00 | | | | /5 |
| Class design | 22:00 | | | | /5 |

## Walk coverage
| # | Item | Hit/Partial/Miss | Evidence |
|---|---|---|---|
| 1 | Actors & entry point | | |
| 2 | Core operations | | |
| 3 | Rules & legality | | |
| 4 | Lifecycle & terminal states | | |
| 5 | Failure behaviour | | |
| 6 | Multiplicity & domain variants | | |
| 7 | Concurrency posture | | |
| 8 | Explicit Out of Scope | | |

**Budget allocation:** <how the 22 minutes actually split>
**First-pass completeness:** <what message two added, per phase; anything back-filled late>
**Silent assumptions:** <what he designed against without asking, and what it would have cost>
**Dangling rules:** <rules raised without a resolution>
**Untraceable state / unenforceable requirements:** <fields no requirement needs; requirements no state can enforce>

## What he produced (verbatim, as it stood at 22:00)
### Requirements
### Out of Scope
### Entities & relationships
### Class design

## What was still missing at 22:00

## Where the time went

## Ideal front half (writable in the same 22 minutes)
### Requirements
### Entities & relationships
### Class design

## Feedback given
```

## Weaknesses file

Update `lld_weaknesses.md` — read it, then update only `## Requirements & Scoping`, `## Entity Modelling`, `## Class Design & Encapsulation`, `## Responsibility Placement`, and `## Communication & Pace`. Increment Sessions on recurring rows, add new ones, max 5 per category (replace only if the new entry is more severe or more specific; labels under 10 words). Decay rows the drill gave a real opportunity to exhibit and he didn't — this drill always exercises requirements, entities, and class design, so those decay whenever he doesn't show them. Refresh only the **Scopes before designing**, **State derived from requirements**, **Rules live with their state (Tell, Don't Ask)**, and **Simplicity held under pressure** rows in `## Senior Signals`; leave the other two and all other tables untouched. Update `Last updated:`, then summarise what changed: added, incremented, **decayed**, **retired**.

## Commit & push

1. `git add lld_weaknesses.md lld_senior_guidance.md transcripts/ .claude/commands/*.md`, then `git status --short` and confirm the new transcripts appear.
2. `git commit -m "LLD sprint: <Problem A>, <Problem B> (front half X/5)"`, ending with the standard co-author line.
3. `git push`. On failure do NOT retry blindly — report exactly what failed and stop.
4. Nothing to commit → say so, skip rather than making an empty commit.

---

**Start now:** stamp the start time, state the format in three lines (requirements + entities + class design, no method bodies; 22:00 is the hard limit for all three; the 6:00 and 10:00 marks are pace targets he may overrun at his own cost, and nothing is discarded for arriving late), present the first problem with its **Difficulty: X/5** line, ask if he has any questions, and start the clock.
