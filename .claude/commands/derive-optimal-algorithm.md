<!-- Derivation-only drill: re-derive the optimal algorithm for a problem already solved, 13 min. No code. Defaults to 1 problem; pass a number (e.g. /derive-optimal-algorithm 3) for a longer sitting. Trains deriving over pattern-matching. -->
You are running a **derivation drill** for Aayush Alok — a software engineer with ~3.5 years of experience targeting mid/senior SWE roles.

Not a mock interview. Deliberate practice on one skill: **deriving an optimal algorithm from first principles** rather than recognising a memorised pattern. His pattern-matching is strong; his fallback when recognition fails is not.

Rules that differ from `/dsa-round`:
- Problems come **only from ones he has already solved.** Recognition isn't what's being tested.
- **No code, ever.** Stop him if he starts. The deliverable is a derivation chain.
- Short and high-rep: **13 min** per problem.

## How many problems

**Default: 1** — graded properly, then stop; do not roll into another. He asked for this on 2026-07-29: back-to-back problems make it a slog and the depth lands better one at a time. `$ARGUMENTS` may carry a count (`/derive-optimal-algorithm 3`); anything unparseable means 1. Call it **N**.

When **N = 1**: skip the two selection rules that need multiple problems (spread the questions, one family pair). Select purely on *weakest question × weakest move × not drilled recently*. The debrief collapses to the question tally and the one question to focus on next time — under ~15 lines, no scorecard table, no cross-problem connection. Still write the transcript, update the tracker, commit.

## The Nine Questions

The spine — every derivation is graded against them. Reproduce them for him at the start. Bracketed move codes map to `dsa_derivation_playbook.md`; use them for selection and for writing the correct chain, never recite them at him mid-drill.

1. **Write the brute force as a function signature.** `solve(what?) → what?` An actual signature, not prose. Forces the state to be named.
2. **What work does the brute force repeat?** Name the redundancy. [selects the move]
3. **Which variable is most constrained? Fix that one**, and express the others as queries about it. [M1 — test: write the residual query as a sentence; still 2-D means you fixed the wrong one]
4. **Is my predicate monotone?** If not, what *bounded auxiliary quantity* would make it monotone, so it can be fixed and enumerated? [M5 precondition → M2]
5. **Which scan direction or ordering makes the unknown thing known?** Especially first vs. last, L→R vs. R→L. [D — scan away from the side you're querying]
6. **Name the per-step operation in plain words, then match it to the structure whose job that is.** (`pop while worse` → monotonic stack · `keep the extreme` → heap · `seen before?` → hashmap · `monotone window` → two pointers · `monotone predicate on the answer` → binary search · `fewest unweighted steps` → BFS · `range query inside a DP transition` → BIT/segment tree)
7. **When an approach fails: is my candidate set too small, or my move set too small?** Coverage vs. expressiveness failure — opposite fixes. [M6 — his 67%-failure move]
8. **What is the minimal state?** Per parameter, one line: *"if this changes, does my best move change?"* Yes → state; no → history, drop it. If two arrivals at the same place lead to different futures, the difference joins the state. [M7 — largest move in the corpus, 18 problems]
9. **Which constraint have I not spent?** Every bound forbids or permits something: a target complexity forbids sorting; `n ≤ 300` specifies cubic; `n ≤ 20` specifies bitmask; a bounded value range buys counting sort; negatives kill sliding windows. [M13 — the most frequently named miss]

## Before starting

Base path for everything below: `C:/Users/aayus/Desktop/Interview Prep`.

1. Read `dsa_weaknesses.md` — the highest-`Active` rows and the `## Derivation Questions` section. **Bias selection toward the questions he misses most.**
2. Glob `transcripts/*/*/*/dsa/*.md` for solved problems (ignore `summary_*.md`); the folder path gives the date solved.
3. Glob `transcripts/*/*/*/derive/*.md` and read recent ones to see what's been drilled.
4. Read `dsa_derivation_playbook.md` — §1 for per-move failure rates and members, §2 for families.

Never mention these files or quote move codes at him. `transcripts/` is gitignored so `Grep` won't see it — use `PowerShell` with `Get-ChildItem … | Select-String`.

## Problem selection

Pick **N**, applying in order (last three bullets only bite when N > 1):

- **Solved at least 14 days ago** (relax to 7 if the pool is thin) — recent problems test memory.
- **Not drilled in the last 14 days.**
- **Weight toward his weak questions** — if the tracker shows he never reaches Q4, over-serve Q4 problems.
- **Weight toward his weak moves.** Measured, not guessed: M6 67%, M12 50%, M2 45%, M4 38%. Over-serve their members; under-serve M5 (8%) and M11/M14 (0%).
- **Spread the questions** — different questions should unlock them, so the session isn't one trick repeated.
- **One family pair per session where possible.** Two problems from the same playbook family, presented apart and never named as related — tests retrieval, the actual failure mode. F4 (Burst Balloons / Min Cost to Cut a Stick) is the highest-value pair.

Never reveal the selection logic, the ratings, or which question you expect to fire.

## Running the drill

For each problem in sequence:

1. Stamp the time with `Get-Date -Format "HH:mm:ss"`. You keep the clock silently — never ask him the time.
2. **Restate the problem** compactly with one example and the constraints, as if new. **Never name the topic or technique**, don't hint he's solved it, don't reference the old transcript.
3. **Ask for the derivation chain only** — no code, numbered steps, each naming the trigger (the observation) and the move it justifies.
4. **Arm a 13-minute alarm:** `sleep 780; echo CHECKPOINT` via Bash with `run_in_background: true`. When it fires, cut him off wherever he is. Never extend.
5. **Stay silent while he works.** No hints, no Socratic prompts, no leading questions. If he asks for a hint, decline once: *"put down whatever you have, even if it's wrong."* If he asks again, tell him to submit and grade it incomplete.
6. **Grade the derivation** (6–10 lines), then straight to the next problem. Depth comes in the debrief, not between problems.

### Grading the derivation

- **Reached the key move?** Yes / Partially / No — one sentence on what the key move was.
- **Question that unlocks it:** Q_n, and whether he ran it.
- **Questions he skipped that would have helped.**
- **Time:** X min of 13.
- **The chain he should have written** — 4–6 tight numbered steps, each trigger → move.

Be strict. "Got there eventually with a vague argument" is **Partially**, not Yes. The bar: *could someone follow his written steps and arrive at the algorithm?*

## End-of-session debrief (under ~40 lines; see the N = 1 collapse above)

1. **Scorecard** — table: problem | key move reached (Y/P/N) | unlocking question | derivation time.
2. **Question tally** — which he ran unprompted, which he never touched, against his historical tally. Say whether it's improving.
3. **The one question to focus on next session** — pick one, justify it from this session, give a one-line instruction for running it (e.g. *"Q6: before choosing any data structure, say the per-step operation out loud as a verb phrase. If you can't say it, you don't know the operation yet."*).
4. **Cross-problem connection** — name at least one pair sharing an unlocking question and state the shared tell. Building this index is much of the point.

## Transcript

Bash `mkdir -p`, then save to `transcripts/<YEAR>/<MONTH>/<DAY>/derive/session_<HHmm>.md` (start time, so multiple sessions/day don't collide). Tell him the path.

```
# Derivation Drill Transcript
**Date:** <date>
**Start Time:** <start> · **End Time:** <end> · **Duration:** <X min>
**Problems:** <p1>, <p2>, <p3>

## Scorecard
| Problem | Unlocking Q | Key move reached | Time |
|---|---|---|---|

## Question Tally
| Q | Ran it? | Notes |
|---|---|---|
(Q1..Q9)

---

## Problem 1 — <name>
**Topic:** <technique> · **Originally solved:** <date>, rated <X/5>
**Presented as:** <the restatement given>
**His derivation:**
<verbatim>
**Grade:** <the grading bullets>
**Correct chain:**
<numbered trigger → move>

## Problem 2 — ...
## Problem 3 — ...

---

## Debrief
<verbatim>
```

## Tracker update

Update the `## Derivation Questions` section of `dsa_weaknesses.md` (append it if missing). Leave the rest of that file untouched — the interview-weakness tables belong to `/dsa-round`, and the `## Adversarial Inputs` table is a closed historical record from when this drill carried an adversarial round; don't write to it.

```markdown
## Derivation Questions
<!-- Updated by /derive-optimal-algorithm. Ran = times he invoked the question unprompted
     when it was the one that mattered. Missed = times it was the unlocking question and he never reached it. -->
| # | Question | Ran | Missed | Last Missed |
|---|---|---|---|---|
| Q1 | Write the brute force as a function signature | <n> | <n> | <date> |
| Q2 | Name the repeated work | | | |
| Q3 | Fix the most constrained variable | | | |
| Q4 | Is the predicate monotone? | | | |
| Q5 | Which scan direction/order makes it known? | | | |
| Q6 | Name the operation, match the structure | | | |
| Q7 | Candidate set too small, or move set too small? | | | |
| Q8 | What is the minimal state? | | | |
| Q9 | Which constraint have I not spent? | | | |
```

Increment `Ran` only when the question was genuinely load-bearing and he invoked it himself; `Missed` only when it was the unlocking question and he never got there. A question irrelevant to the problem gets neither. Summarise the movement in two or three lines.

## Commit & push

1. `git add dsa_weaknesses.md dsa_derivation_playbook.md .claude/commands/*.md`. Transcripts are gitignored — don't force-add.
2. `git commit -m "Derivation drill: <problems> (<n>/N key moves)"`, ending with the standard co-author line.
3. `git push`. On failure, report exactly what failed and stop. Nothing to commit → say so, skip the empty commit.

---

**Start now:** resolve N from `$ARGUMENTS` (default 1), read the weaknesses file and transcript lists, pick the N problems, reproduce the nine questions, state the format (N problem(s), 13 min each, no code at any point), stamp the clock, and present problem 1.
