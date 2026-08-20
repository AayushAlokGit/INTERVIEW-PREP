# LLD Round Transcript
**Date:** 2026-08-20
**Start Time:** 11:37:09 · **End Time:** 12:55:29 · **Duration:** 78 min
**Problem:** Spreadsheet Cell Dependencies (cell and formula engine)
**Category:** infrastructure object
**Difficulty:** 5/5 — the object model has to carry a dependency graph, a cycle check that cannot mutate speculatively, and a recompute ordering that a naive traversal gets wrong; three independent hard things inside a 40-minute box
**Performance Rating:** 3/5  <!-- machine-read on future rounds; <=2 = eligible for re-ask, >=3 retired -->
**Hints Used:** 1/2
**Requirements Asked:** core operations · rules & legality for both verbs · multiplicity (one spreadsheet vs many, cell count) · **Never Asked:** actors & entry point · lifecycle (clear/overwrite/undo) · failure convention (throw vs return) · concurrency posture · whether a value can be non-numeric
**Walk coverage:** 4/8 hit (1 partial, 3 miss)
**Out of Scope list produced:** Unprompted (thin — UI, durability)
**Self-Verified:** No — traced only after being asked, and his claimed trace was wrong on every line (dirty set, in-degrees, queue order, and all three values; claimed readCell("C1") = 11 where his own code yields 21)
**Concurrency follow-up:** correctness — named the category before the primitive, chose a coarse lock, stated the throughput cost, and named fine-grained locking with its total-ordering/deadlock precondition. Strongest answer of the round
**Would it have fit a real 40-min round?** No — cut off during class design (~20 min), with no parse, no dependents map, and no setCell signature yet

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements + Out of Scope | 5 min | 8 min | +3 | Close |
| Entities & relationships | 8 min | 21 min | +13 | No |
| Class design | 20 min | 39 min | +19 | No |
| Implementation + trace | 32 min | 64 min | +32 | No |
| Extensibility + concurrency | 40 min | 78 min | +38 | No |
| **Total** | **40 min** | **78 min** | **+38** | **No** |

---

## Problem Statement

Design the cell and formula engine of a spreadsheet.

**Difficulty: 5/5**

Five phases announced (requirements + out of scope · entities & relationships · class design · implementation + trace · extensibility + concurrency), with the 40-minute reference timeline stated as measured but not enforced. No UML. Pseudo-code or a real language, his choice.

---

## Conversation Log

**Interviewer:** [problem statement, difficulty, five phases, reference timeline] Which do you want to write in phase 4 — pseudo-code, or a real language? Any questions before you start?

**Aayush:** What are the operations expected from the system?

**Interviewer:** Two operations. (1) Set a cell's content — the caller gives a cell reference and a content string; that content is either a literal number, or a formula that references other cells. (2) Read a cell's value — the caller gives a cell reference and gets back the current computed value. That's the whole public surface. Still need your answer: pseudo-code or a real language?

**Aayush:** What are the rules for setting a cell's content and reading value?

**Interviewer:** Content is a number literal ("5", "-2.5") or a formula starting with `=`, combining cell references and number literals with `+` and `-`. Legality: (1) the formula must parse — malformed is illegal; (2) a formula may reference a never-set cell, treated as 0; (3) a set that would create a circular reference is illegal, and the spreadsheet must be left exactly as it was before the call. Reading returns the current computed value as a number; a never-set cell reads as 0; after any successful set, every cell's value must be correct on the next read. Third time: pseudo-code or a real language?

**Aayush:** pseudocode. Will the engine manage only one spreadsheet or multiple spreadsheets and how many cells in a spreadsheet?

**Interviewer:** One spreadsheet per engine instance, no cross-spreadsheet references. The grid is nominally A–ZZ by 1–1,000,000 but sparse — only cells actually set exist. On the order of 100k populated cells at the top end, all in memory.

**Aayush:** [numbered requirements list restating operations, rules, and multiplicity, followed by] Out of scope -> 1. UI 2. Durability

**Interviewer:** Recorded. How do you want to start?

**Aayush:** Entities -> SpreadsheetEngine(Map<string,Cell>) · Cell(value:string, row:string, column:int)

**Interviewer:** Three questions. (1) `Cell` holds `value: string`, but read returns a number. When I call `read("A1")` and A1 was set to `"=B1+2"`, which entity computes the number, and what is in `value` at that moment? (2) You have `row: string, column: int` — walk me through what they hold for cell B7. (3) Rule (c): a set that would create a circular reference is illegal and must leave the spreadsheet unchanged. Which of these two entities knows that A3 depends on A1?

**Aayush:** 1. Cell will hold value as double and another formula:string attribute. When setting the cell's value the formula will be evaluated to determine the value. 2. Row:"B" Col:7. 3. Spreadsheet engine will have the knowledge of dependencies. While evaluating formula the engine can save a dependency graph of cells.

**Interviewer:** Two follow-ups. (1) The dependency graph — say concretely what one entry maps to what. If `A3 = "=A1+A2"`, what does the graph contain after that set? (2) You said the engine saves the graph *while evaluating the formula*. Suppose `A1 = "=A3+1"` is already set and I call `set("A3", "=A1+2")` — the illegal circular case. If evaluation is what builds the graph, what state is the graph in at the moment you discover the cycle?

**Aayush:** 1. It will be a directed graph with edges from A1 and A2 to A3. 2. After evaluation the dependencies can be calculated and the graph can be updated by checking if updating creates a cycle or not; if it does, then the formula is invalid.

**Interviewer:** Number the steps for that exact call — `A1` is already `"=A3+1"`, I call `set("A3", "=A1+2")`. Step 1, step 2, step 3, until you return. Then take me into class design.

**Aayush:** 1. Evaluate the formula string to extract the cell references (those which start with a letter followed by a number without space). 2. Check if adding an edge from the dependency cells to the current cell being set creates a cycle or not. If cycle created, throw exception. 3. If no cycle created, add the edges to the graph and then set the value of the cell, and also the formula string for the cell.

**Interviewer:** Good — that's an ordering I can hold you to. Now class design: both classes, state with types, public methods with full signatures and return types.

**Aayush:** [class Cell with value/formula/row/column, setValue/setFormula/getValue/getFormula; class SpreadsheetEngine with cellMap, dependencyGraph: Map<string,List<string>>, validateContentString, setCell(row,column,content), readCell(row,column)]

**Interviewer:** Four questions. (1) For `A3 = "=A1+A2"`, spell out the actual `dependencyGraph` entries — what key, what list? (2) `Cell` carries `row` and `column`, but `cellMap` is keyed by `"A1"` and both public methods take row and column separately. Which requirement needs the cell to store them? (3) `getFormula()` — who calls it, and what do they do with the string? (4) Run this and tell me what comes back: `setCell("A",1,"5")` · `setCell("A",3,"=A1+2")` · `setCell("A",1,"9")` · `readCell("A",3)`.

**Aayush:** 1. A1 -> A3, A2 -> A3. 2. No requirement needs row and column to be stored separately; the cell reference string can simply be stored and passed around in setCell and readCell. 3. The method is present if we wish to fetch the formula applied to a cell instead of the computed value. Not needed according to requirements. 4. SpreadsheetEngine needs another method evaluateFormula which takes a formula string and the cell map and computes the current value according to the formula. This method will be called while setting the value for the cell and also when reading a cell with a formula, to verify if Cell.value needs to be updated or not. With these changes readCell(A,3) will give 11.

**Interviewer:** Two things. (1) Same style of sequence, one level deeper — what comes back? `setCell("A",1,"5")` · `setCell("A",2,"=A1+1")` · `setCell("A",3,"=A2+1")` · `setCell("A",1,"9")` · `readCell("A",3)`. (2) Then write phase 4 in pseudo-code: `setCell`, `readCell`, `evaluateFormula`, plus revised signatures.

**Aayush:** 1. When updating a cell, start a BFS from this cell to also propagate the cell value update to the other cells which depend on it. This way in setCell all the relevant cell values will be updated once a cell has been updated. No operation required at read time. At write time the updates are propagated; this makes read very simple.

**Interviewer:** Write it. Pseudo-code for `setCell`, `readCell`, and `evaluateFormula`, plus the revised signatures.

**Aayush:** update md file

**Interviewer:** [sheet updated] Now the code.

**Aayush:** [submits revised outlines: `Term`, `Parsed`, `Cell` with value/formula/terms/refs, `SpreadsheetEngine` with cellMap + dependents, and prose descriptions of parseContent, evaluateFormula, currentValue, reachesAny, attachEdges, propagate (Kahn), normalizeRef, isInBounds]

**Interviewer:** That's outlines and prose descriptions, not implementations. Write the actual pseudo-code bodies for `setCell`, `propagate`, and `reachesAny`. Statements, not commentary.

**Aayush:** [submits setCell — normalize, bounds check, parse phase, legality phase (self-ref + reachesAny), mutate phase (getOrCreate, attachEdges, assign terms/refs/formula/value, propagate)]

**Interviewer:** Now `propagate` and `reachesAny`.

**Aayush:** [submits propagate — DFS to collect dirty, in-degree over precedents inside dirty, Kahn recompute, assert done == size(dirty)]

**Interviewer:** `reachesAny`.

**Aayush:** [submits reachesAny — DFS forward over dependents from target, return true if any visited node is in targets]

**Interviewer:** Walk one scenario through it, step by step, naming state after each step: `setCell("A1","1")` · `setCell("B1","=A1+1")` · `setCell("C1","=A1+B1")` · `setCell("A1","10")` · `readCell("C1")`. Break out the `propagate("A1")` call — `dirty`, `indeg`, queue order, and each `c.value` as written.

**Aayush:** dirty -> {A1, B1, C1} indeg -> A1-0 B1-1 C1-2 / queue -> {A1} -> {B1} -> {C1} / A1 c.value = 9, B1 c.value -> 10, C1.value -> 11

**Interviewer:** Noted — taking that as your result. Follow-up one, verbal only. We now need range functions: `"=SUM(A1:A10)"` and `"=MAX(B1:B4)"`. Where does that land — which existing pieces change, and which don't?

**Aayush:** Currently all terms are just used for summation. With ranges the parseExpression will now parse the range to all relevant refs, and it will also output the operation to be performed on the value of the dependent cell refs (currently just summation). The cell will also have another attribute for tracking the aggregation operation — MIN, MAX, SUM etc.

**Interviewer:** Then give me `"=SUM(A1:A3)+MAX(B1:B2)-4"` under that scheme. What is in `cell.terms`, and what is in your new aggregation attribute?

**Aayush:** Now terms will not only be refs or numeric literals — they are also expressions which can be terms. And expressions will have the aggregation type and their own refs involved.

**Interviewer:** Follow-up two. We add division, so `"=A1/B1"` is legal. If `B1` is 0 the cell's value is `#DIV/0!`, and any cell referencing it must show `#REF!` instead of a number — errors propagate through the dependency graph the same way numbers do. What has to change?

**Aayush:** The changes should be in evaluateFormula, checking for division with 0 if it is occurring.

**Interviewer:** `readCell(ref) -> double` and `Cell.value: double`. What does `readCell("A1")` hand back when A1 is `#DIV/0!`, and what does `evaluateFormula` do with that when it hits `A1` inside `"=A1+5"`?

**Aayush:** give hint

**Interviewer:** [HINT 1/2] The pressure is on your **value type**, not on `evaluateFormula`'s arithmetic. `double` can represent a number and nothing else — an error is a second kind of cell value that has to survive being stored, returned, and consumed as an input to another formula. Decide what `Cell.value` and `readCell`'s return type become, and where the rule *"if any of my inputs is an error, my value is an error"* lives.

**Aayush:** readCell will return either double or exception. And the rule of applying exception to computed values should live in evaluateFormula.

**Interviewer:** Last follow-up. The engine is now shared: several threads call `setCell` on the same spreadsheet concurrently. Thread 1 calls `setCell("A1","5")`; at the same moment Thread 2 calls `setCell("B1","=A1+1")`. What breaks, and what would you do about it?

**Aayush:** The final value of cell B1 could be 1 instead of expected 6. This is a correctness issue; it is arising because there is no serialisation around accessing the shared cellMap and dependency maps. A solution is a coarse lock around setCell, acquired at the beginning and held throughout to the end. This will reduce throughput of writing to the spreadsheet. An alternative is fine-grained locking by acquiring locks on cells, but then we would need to impose a total ordering for acquiring locks, else we risk deadlock.

---

## Walk coverage (phase 1)

| # | Item | Hit/Partial/Miss | Evidence |
|---|---|---|---|
| 1 | Actors & entry point | Miss | Never asked who calls the engine; `SpreadsheetEngine` appeared as an entity without being named as the entry point |
| 2 | Core operations | Hit | "what are the operations expected from the system?" |
| 3 | Rules & legality | Hit | "what are the rules for setting a cell's content and reading value?" |
| 4 | Lifecycle & terminal states | Miss | Never asked whether a cell can be cleared, whether a set is an overwrite, whether undo exists |
| 5 | Failure behaviour | Partial | Never asked the convention; chose `throw` unilaterally at step 2 of the ordering, then held it consistently across every signature |
| 6 | Multiplicity & domain variants | Hit | "will the engine manage only one spreadsheet or multiple and how many cells" |
| 7 | Concurrency posture | Miss | Never asked; surfaced only because the curveball was served |
| 8 | Explicit Out of Scope | Hit | Produced unprompted: UI, durability |

**Dangling rules:** none — every rule he raised got a resolution, including the cycle-check ordering once pushed on.

**Silent assumptions:**
- *A set is always a full overwrite; there is no clear/delete.* `attachEdges` happens to cover it, but by luck rather than decision — asking would have made `clear(ref)` an explicit API decision.
- *Values are always numeric.* Directly caused the phase-5 collapse: `value: double` and `readCell -> double` were locked in before he asked whether a cell could hold anything else. That is the exact field the error follow-up broke, and why the hint was spent there.
- *Single-threaded.* Correct assumption, never stated — one sentence would have earned the credit.

---

## His Design

**Requirements he gathered:** operations (set content, read value) · content rules (number literal or `=`-formula over refs and literals with `+`/`-`) · legality (must parse, never-set ref is 0, no circular reference and the sheet left unchanged) · read semantics (never-set reads 0, all values correct on next read) · multiplicity (one spreadsheet, sparse A–ZZ x 1–1,000,000, ~100k cells in memory).

**Out of Scope:** UI · durability.

**Entities & relationships:** initially `SpreadsheetEngine(Map<string,Cell>)` and `Cell(value:string, row:string, column:int)`; corrected under questioning to `Cell(value:double, formula:string)` with the engine owning a directed dependency graph (`A1 -> A3`, `A2 -> A3` for `A3 = "=A1+A2"`), then to `Cell(value, formula, terms, refs)` with the engine owning `cellMap` and `dependents`, plus `Term` and `Parsed` structs.

**Class outlines:** final form is `Term`, `Parsed`, `Cell(value/formula/terms/refs; getValue/getFormula)`, `SpreadsheetEngine(cellMap, dependents; setCell/readCell public, parseContent/evaluateFormula/currentValue/reachesAny/attachEdges/propagate/normalizeRef/isInBounds internal)`. Dropped `row`/`column` when asked which requirement needed them; dropped `getFormula` as not required. Full outlines on the design sheet.

**Core implementation:** `setCell` (parse -> legality -> mutate, mutate cannot throw), `propagate` (DFS dirty set excluding start, in-degree over precedents inside dirty, Kahn recompute, assert), `reachesAny` (forward DFS over `dependents` from target). Full pseudo-code on the design sheet.

**Gaps / misplaced responsibilities:**
- `Cell.value: double` and `readCell -> double` — the one field that outlived its justification, and the seam the error follow-up broke.
- The error rule placed in `evaluateFormula` as a divide-by-zero guard, where a value-type decision belonged.
- The aggregation attribute first placed on `Cell`, which `"=SUM(A1:A3)+MAX(B1:B2)-4"` immediately broke; re-homed into a recursive `Term` under one counterexample.
- No unprompted trace; the prompted trace contradicted his own code's stated contract on every line.
- Phases 2 and 3 arrived as sketches that were rebuilt three times through questioning rather than modelled once.

---

## Feedback Given

**Round conditions.** 1 hint used (the value-type seam on error propagation) -> ceiling 3/5. Asked: operations, rules, multiplicity. Never asked: actors, lifecycle, failure convention, concurrency posture. Out of Scope produced unprompted but thin. Did not self-verify; the prompted trace was wrong.

**Rubric.**

- **Requirements & scoping — 3/5.** Three of eight items missed, which caps it. What he did ask was well-aimed — operations, then rules, then multiplicity is the right spine — but asked one per turn across four messages, which reads as curiosity rather than a walk.
- **Entity modelling — 4/5.** The final model is right: `Cell` owns its parsed state, the engine owns the graph and orchestrates. He got there by correction — the opening `Cell` had a string where a number belonged, an inverted row/column, and no representation of the dependency at all.
- **Class design — 4/5.** `Term`/`Parsed` as a parse result returned but never stored is a real decision; the comment that `dependents` keys may name never-set cells is senior-grade. Points off for a first submission carrying four accessors with no caller identified.
- **Responsibility placement — 4/5.** `evaluateFormula(terms)` rather than `(string, cellMap)` is the correct boundary — recompute never re-parses. The one leak: the error rule placed as a divide-by-zero guard in `evaluateFormula`, where a value-type decision belonged.
- **Implementation & correctness — 2/5.** The code is correct; the trace was not. He claimed `dirty = {A1,B1,C1}`, `indeg` A1-0/B1-1/C1-2, queue A1->B1->C1, values 9/10/11, `readCell("C1") = 11`. His own code gives `dirty = {B1,C1}` — his own comment reads "start excluded — already fresh" — `indeg` B1-0/C1-1, queue B1->C1, and values A1=10, B1=11, C1=**21**. A 9 appears nowhere in the scenario. Worse: had `dirty` actually contained A1 as claimed, propagate would call `evaluateFormula` on a literal cell whose `terms` is empty, summing to 0 and silently zeroing A1. His code is right precisely because it excludes the start; his trace asserted the version that would have broken it.
- **Simplicity & judgement — 4/5.** Zero patterns, none needed. He cut `row`/`column` and `getFormula` the moment they could not be justified. The three-phase `setCell` — parse, check, mutate, "mutate cannot throw, hence no rollback path" — is the best decision in the design: it turns rule (c) from a rollback problem into a structural one.
- **Extensibility — 3/5.** Ranges landed cleanly at the parser seam. The cell-level aggregation attribute broke immediately on `"=SUM(A1:A3)+MAX(B1:B2)-4"` and he re-homed it into a recursive `Term` under pressure without being told — a genuine save. Errors is where it stopped: "check for division by zero in evaluateFormula" does not engage the problem, and even after the hint, "readCell returns either a double or an exception" leaves open what is *stored* in `Cell.value` and what propagate writes there.
- **Concurrency — 5/5.** The strongest answer of the round. Named the category (correctness) before any primitive, identified the mechanism (no serialisation around the shared `cellMap` and dependents map), chose the coarse lock as the smallest thing that works, stated its cost (write throughput), and named the fine-grained alternative with its precondition — total lock ordering or deadlock.
- **Communication — 3/5.** Reasoning visible, and every counterexample produced a real correction rather than a defence. Two costs: the language question took three asks, and two direct "what comes back?" questions were answered with design changes instead of a value.

**Pace report.** 78 minutes against a 40-minute reference — nearly double. A real interviewer cuts him off during **class design**: at the 20-minute mark he had `Cell(value:string, row:string, column:int)` and a one-line engine — no parse, no dependents map, no `setCell` signature. Everything that makes this design good arrived after minute 39; `propagate`, the Kahn ordering, the three-phase set and the top-scoring concurrency answer would never have been reached. This was a 5/5 problem and **the overrun was process, not size**. Two sinks: phase 2 took 21 minutes because entities arrived as a two-line sketch and were then rebuilt three times through questioning; and the same design was re-derived three times — BFS at minute 39, topological at minute 58, correct code after. The topological insight was available at minute 39 had he traced the diamond scenario instead of answering it with a strategy.

**Performance Rating: 3/5.** This would have been a 4 — the final artifact is strong-hire quality — **capped at 3 by the one hint used**. The wrong trace independently pins Implementation at 2 and would have held the score at 3 regardless.

**Senior-signal scorecard.**

| Signal | Status | Reason |
|---|---|---|
| Scopes before designing | Mixed | Right three questions, asked one per turn; missed actors, lifecycle, failure convention, concurrency posture |
| State derived from requirements | Strong | Cut `row`/`column` and `getFormula` on demand; every surviving field traces to a rule. `value: double` is the one field that outlived its justification |
| Rules live with their state (Tell, Don't Ask) | Strong | `evaluateFormula(terms)` not `(string, cellMap)`; the freshness invariant sits on `Cell`, enforced by its owner |
| Simplicity held under pressure | Strong | Zero patterns, none missed. Three-phase set eliminates rollback rather than implementing it |
| Verifies own logic | Weak | No unprompted trace; the prompted one contradicted his own code's stated contract on every line |
| Extends without rewriting | Mixed | Ranges landed at the parser seam and the `Term` recursion was a genuine save; errors needed a hint and still did not resolve |

**Overall read: senior on artifact, mid-level on process. Hire — not strong-hire.** The design he ended with is one I would ship. But a senior strong-hire produces it inside the box and can execute it on paper. He produced it at minute 78 and then could not run four lines of his own pseudo-code correctly. In a real loop the first costs the round; the second costs the level.

**What a senior strong-hire would have done on this problem.**
- Asked "can a cell hold text, or only numbers?" in phase 1 — fifteen seconds, and the answer that broke him in phase 5. Knowing values are a closed set of kinds up front, `Cell.value` is a `CellValue` (number | error) from minute 3, `readCell -> CellValue`, and the error follow-up costs one sentence instead of a hint.
- Asked the failure convention explicitly rather than letting `throw` emerge at step 2 of an ordering discussion. He held it consistently, so he got the outcome without the credit.
- `Cell.value: double` is the one field in the wrong shape; the rule "an error input makes me an error" belongs with that value type, not as a divide-by-zero guard inside `evaluateFormula`.
- The aggregation attribute on `Cell` was the one wrong home — aggregation is a property of a sub-expression, not of a cell, since one cell can contain two different aggregates. He corrected it in a single turn under a counterexample; a strong-hire feels the mixed-expression case before stating the design.
- The edge case his own trace would have caught: `propagate` calling `evaluateFormula` on a literal cell with empty `terms`, returning 0. His code never does this because `dirty` excludes the start — but his trace asserted that it did, which means he did not know why his code was right. The diamond scenario at minute 34 was the moment to trace, and he answered it with a strategy instead.
- No pattern was the right answer here, and he took it. Worth naming: an Observer for recompute would have been the classic overreach — the dependents map *is* the observer registry, and formalizing it hides the ordering guarantee the design depends on, since Observer notifies in registration order while correctness requires topological order.

**Drill.** Take the code exactly as written. On paper, no editor, run `setCell("A1","2")` · `setCell("B1","=A1+1")` · `setCell("C1","=A1+B1")` · `setCell("A1","7")` and write the full state of `cellMap`, `dependents`, `dirty`, `indeg`, and the queue after **each** call. Then the illegal one — `setCell("A1","=C1+1")` — and write what `reachesAny("A1", {"C1"})` visits, in order, and prove `cellMap` and `dependents` are byte-identical afterwards. Repeat until the numbers come out right the first time.

**Checklist.** `lld_senior_guidance.md` — two lines are the whole story of this round: "Ran all eight items of the requirements walk" and "Traced one concrete scenario out loud, naming state changes."
