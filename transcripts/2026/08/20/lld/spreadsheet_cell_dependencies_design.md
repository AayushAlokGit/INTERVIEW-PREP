# Spreadsheet Cell Dependencies

## Phase 1 — Requirements

1. **Operations**
   - Set a cell's content — the caller gives a cell reference and a content string. That content is either a literal number, or a formula that references other cells.
   - Read a cell's value — the caller gives a cell reference and gets back the current computed value.

2. **Rules**

   Setting content. The content string is one of two things:
   - a number literal — `"5"`, `"-2.5"`
   - a formula — starts with `=`, an expression combining cell references and number literals with `+` and `-`. E.g. `"=A1+B2-3"`.

   Legality rules for a set:
   - a. The formula must parse. A malformed formula is illegal.
   - b. A formula may reference a cell that has never been set — that reference is treated as 0.
   - c. A set that would create a circular reference is illegal, and the spreadsheet must be left exactly as it was before the call.

   Reading a value. Returns the cell's current computed value as a number. A cell that has never been set reads as 0. After any successful set, every cell's value must be correct on the next read.

3. One spreadsheet managed by the engine instance. The grid is nominally large — columns A–ZZ, rows 1–1,000,000 — but sparse. Only cells that have actually been set exist. On the order of 100k populated cells at the top end, all in memory.

## Out of Scope

1. UI
2. Durability

## Phase 2 — Entities & relationships

- **SpreadsheetEngine** — holds `Map<string, Cell>`. Owns knowledge of dependencies; saves a dependency graph of cells while evaluating a formula.
- **Cell** — `value: double`, `formula: string`, `row: string`, `column: int`. For cell B7: `row = "B"`, `column = 7`. When setting the cell's value the formula is evaluated to determine the value.

## Phase 3 — Class outlines

```
class Cell:
- value: double
- formula: string|null
- row: string
- column: int
+ setValue(val: double) -> void
+ setFormula(formula: string) -> void   // throws InvalidFormulaException
+ getValue() -> double
+ getFormula() -> string|null

class SpreadsheetEngine:
- cellMap: Map<string, Cell>              // "A1" -> Cell of A1
- dependencyGraph: Map<string, List<string>>
- validateContentString(content: string)
+ setCell(row: string, column: int, content: string) -> void
      // throws InvalidContent, CircularReferenceException
+ readCell(row: string, column: int) -> double
```

**Set ordering he committed to** (for `set("A3", "=A1+2")` where `A1 = "=A3+1"`):
1. Evaluate the formula string to extract the cell references (those which start with a letter followed by a number without space).
2. Check if adding an edge from the dependency cells to the current cell being set creates a cycle. If a cycle is created, throw exception.
3. If no cycle created, add the edges to the graph and then set the value of the cell, and also the formula string for the cell.

**Dependency graph shape:** directed graph; for `A3 = "=A1+A2"` there are edges from A1 and A2 to A3.

### Phase 3 — revisions

**Dependency graph entries** for `A3 = "=A1+A2"`: `A1 -> A3`, `A2 -> A3`.

**`Cell.row` / `Cell.column` dropped:** "no requirement needs row and column to be stored separately; the cell reference string can simply be stored and that can also be passed around in `setCell` and `readCell`."

**`getFormula()`:** "present if in the case we wish to fetch the formula applied to a cell instead of the computed value. Not needed according to requirements."

**New method — `evaluateFormula`:** on `SpreadsheetEngine`. Takes in a formula string and the cell map, and computes the current value according to the formula.

**Recompute strategy:** when updating a cell, start a BFS from this cell to propagate the cell value update to the other cells which depend on it. In `setCell` all the relevant cell values will be updated once a cell has been updated. No operation required at read time — at write time the updates are propagated, which makes read very simple.

## Phase 4 — Revised class outlines (his submission)

```
struct Term
- kind:  REF | NUM
- ref:   string          // set when kind == REF
- num:   double          // set when kind == NUM
- sign:  +1 | -1         // sign carried from the preceding operator

struct Parsed                // return value of the parser; not stored
- isFormula: bool
- literal:   double          // meaningful only when isFormula == false
- terms:     List<Term>
- refs:      Set<string>


class Cell
- value:   double            // INVARIANT: always current, never stale
- formula: string|null       // original text, retained for display only
- terms:   List<Term>        // parsed form; empty for a literal
- refs:    Set<string>       // distinct refs in terms; used to tear down old edges

+ getValue() -> double
      Returns the stored value. No computation — the engine keeps it current.

+ getFormula() -> string|null
      Returns the original formula text, or null if the cell holds a literal.
      Not required by the spec; kept as the natural "formula bar" accessor.


class SpreadsheetEngine
- cellMap:    Map<string, Cell>              // "A1" -> Cell
- dependents: Map<string, Set<string>>       // "A1" -> {"A3"}: recompute these when A1 changes
                                             // keys may name cells that were never set

// ---- public API ----

+ setCell(ref: string, content: string) -> void
      throws InvalidContent, CircularReferenceException
      Three ordered phases: parse (may throw), legality check (may throw),
      mutate (cannot throw). Nothing is written before both checks pass, which
      is what satisfies "left exactly as it was" — no rollback path needed.
      Mutation = detach stale edges, attach new edges, overwrite the cell's
      value/terms/formula, then propagate to dependents.

+ readCell(ref: string) -> double
      Normalizes and bounds-checks the ref, then returns the stored value.
      O(1). A never-set cell reads as 0.

// ---- internal ----

- parseContent(content: string) -> Parsed
      Single pass over the string. Rejects empty input, non-numeric literals,
      out-of-bounds refs, consecutive operators, consecutive operands, and a
      trailing operator. Returns terms + distinct refs so callers never re-parse.

- evaluateFormula(terms: List<Term>) -> double
      Sums the signed terms, resolving each REF through currentValue.
      O(#terms), no recursion. Correct only because every stored value is current
      — takes the parsed terms, not a string, so recompute never re-parses.

- currentValue(ref: string) -> double
      Value of ref, or 0.0 if the cell has never been set.

- reachesAny(target: string, refs: Set<string>) -> bool
      True if any ref is reachable from target by walking dependent edges,
      i.e. already transitively depends on target. Only incoming edges to target
      change during a set, so this can be answered against the live graph with no
      speculative mutation. Self-reference is checked by the caller beforehand.

- attachEdges(ref: string, newRefs: Set<string>) -> void
      Removes ref from dependents[r] for each r in the cell's old refs, dropping
      keys whose set empties; then adds ref to dependents[r] for each new ref.
      Runs for literals too, so a formula -> literal change detaches cleanly.

- propagate(start: string) -> void
      Collects the transitive dependent set of start, then recomputes it in
      topological order (Kahn over the induced subgraph, in-degree counting only
      precedents inside the dirty set). Topological — not BFS level order —
      because a plain BFS can evaluate a cell before all its inputs are fresh.
      Asserts every dirty cell was emitted; a shortfall means a cycle escaped
      the phase-2 check.

- normalizeRef(ref: string) -> string
      Uppercases and canonicalizes. Applied at every public entry point.

- isInBounds(ref: string) -> bool
      Column A–ZZ, row 1–1,000,000.
```

## Phase 4 — Core method implementations (pseudo-code)

```
function setCell(ref, content) -> void:
    ref = normalizeRef(ref)
    if not isInBounds(ref): throw InvalidContent

    // --- PHASE 1: parse. May throw. Nothing mutated yet. ---
    p = parseContent(content)

    // --- PHASE 2: legality. May throw. Still nothing mutated. ---
    if ref in p.refs:                    throw CircularReference   // A1 = "=A1+1"
    if p.isFormula and reachesAny(ref, p.refs): throw CircularReference

    // --- PHASE 3: mutate. Cannot throw — hence no rollback path. ---
    cell = cellMap.getOrCreate(ref, new Cell(value = 0.0))

    attachEdges(ref, p.refs)             // detach old, attach new; runs for literals too

    cell.terms   = p.terms
    cell.refs    = p.refs
    cell.formula = p.isFormula ? content : null
    cell.value   = p.isFormula ? evaluateFormula(p.terms) : p.literal

    propagate(ref)
```

```
function propagate(start) -> void:
    // 1. transitive dependent set (start excluded — already fresh)
    dirty = {}; stack = [start]
    while stack not empty:
        cur = stack.pop()
        for d in dependents.get(cur, {}):
            if d not in dirty: dirty.add(d); stack.push(d)

    // 2. in-degree counting ONLY precedents inside dirty
    indeg = {}
    for d in dirty:
        indeg[d] = count of r in cellMap[d].refs where r in dirty

    // 3. Kahn: every input is fresh before its dependent is evaluated
    queue = [d in dirty where indeg[d] == 0]
    done  = 0
    while queue not empty:
        cur = queue.popFront(); done += 1
        c = cellMap[cur]
        c.value = evaluateFormula(c.terms)
        for d in dependents.get(cur, {}):
            if d in dirty:
                indeg[d] -= 1
                if indeg[d] == 0: queue.push(d)

    assert done == size(dirty)      // shortfall => a cycle escaped phase 2
```

```
// Cycle iff some new ref transitively depends on the target,
// i.e. is reachable from target by walking dependent edges.
function reachesAny(target, targets) -> bool:
    seen = {target}; stack = [target]
    while stack not empty:
        cur = stack.pop()
        for d in dependents.get(cur, {}):
            if d in targets: return true
            if d not in seen: seen.add(d); stack.push(d)
    return false
```

## Phase 4 — Self-verification trace (prompted)

Scenario given:
```
setCell("A1", "1")
setCell("B1", "=A1+1")
setCell("C1", "=A1+B1")
setCell("A1", "10")
readCell("C1")
```

His stated trace of `propagate("A1")`:
> dirty -> {A1, B1, C1} · indeg -> A1-0 B1-1 C1-2
> queue -> {A1} -> {B1} -> {C1}
> A1 c.value = 9 , B1 c.value -> 10, C1.value -> 11

## Phase 5 — Follow-ups

**Follow-up 1 — range functions (`"=SUM(A1:A10)"`, `"=MAX(B1:B4)"`):**
> "Currently all terms are just used for summation operation. With allowing for ranges the parseExpression will now parse the range to all relevant refs. And then it will also output the operation that has to be performed on the value of the dependent cell refs (currently just summation). The cell will also have another attribute for tracking the aggregation operation which is to be performed MIN, MAX, SUM etc."

Pressed with `"=SUM(A1:A3)+MAX(B1:B2)-4"`:
> "Now terms will not only be refs or numeric literals, they are also expressions which can be terms. And expressions will have the aggregation type and their own refs involved."

**Follow-up 2 — error values (`#DIV/0!` propagating as `#REF!` to dependents):**
> "The changes should be in evaluateFormula, checking for division with 0 if it is occurring."

**Follow-up 2 continued — after hint (value type is the seam):**
> "readCell will return either double or exception. And the rule of applying exception to computed values should live in evaluateFormula."

**Concurrency curveball — two threads, `setCell("A1","5")` racing `setCell("B1","=A1+1")`:**
> "The final value of cell B1 could be 1 instead of expected 6. This is a correctness issue; it is arising because there is no serialisation around accessing the shared cellMap and dependency maps. A solution to overcome this is to have a coarse lock around setCell. The lock should be acquired at the beginning and held throughout to the end. This will reduce throughput of writing to the spreadsheet. An alternative is fine grained locking by acquiring locks on cells, but then we would need to impose a total ordering for the acquiring of locks else we risk deadlock."

---
---

# Optimal Reference (what a senior strong-hire would design)

> Written after the round. This is the teaching the round withheld — it includes everything above and everything missed.

## 1. Requirements + Out of Scope

**Actors & entry point.** A single in-process client (the UI layer, out of scope) calls one object: `SpreadsheetEngine`. It is the facade; nothing else is public.

**Core operations.**
- FR1 `setCell(ref, content)` — set a cell's content from a string.
- FR2 `readCell(ref)` — read a cell's current value.

**Rules & legality.**
- FR3 Content is either a number literal or a formula — refs and number literals joined by `+`/`-`.
- FR4 A malformed formula is illegal.
- FR5 A reference to a never-set cell resolves to 0.
- FR6 A set that would create a cycle (direct or transitive) is illegal, and the spreadsheet is left byte-identical to its pre-call state.
- FR7 After any successful set, every cell's value is correct on the next read.

**Lifecycle & terminal states.** *(never asked in the round)*
- FR8 A set is a full overwrite — a cell has no terminal state and can be reset any number of times. Changing a formula cell to a literal must tear down its outgoing dependencies.
- FR9 No undo and no clear/delete in v1.

**Failure behaviour.** *(never asked in the round; chosen unilaterally there)*
- FR10 One convention across every signature: **illegal calls throw**. `InvalidContentException` for FR4, `CircularReferenceException` for FR6. Reads never throw.

**Multiplicity & domain variants.**
- FR11 One spreadsheet per engine. No cross-sheet references.
- FR12 Grid nominally A–ZZ x 1–1,000,000, sparse, ~100k populated cells, in memory.
- FR13 **A cell's value is a number or an error.** *(the question never asked — and the one that decides the whole value type)*

**Concurrency posture.** *(never asked in the round)*
- FR14 v1 is single-threaded; the engine is not thread-safe and says so. Documented, not assumed.

**Out of Scope.** UI · durability · undo/redo · multiplication and division (until the follow-up) · ranges and functions · formatting · text values · cross-sheet refs · authorization · multi-user editing.

## 2. Entities & relationships

- **`SpreadsheetEngine`** — *the orchestrator*. Owns `cellMap` and `dependents`. Owns everything that spans more than one cell: parsing, cycle detection, edge maintenance, recompute ordering.
- **`Cell`** — owns one cell's content and computed value. Holds the invariant *"my value is never stale"*; it cannot enforce it (it cannot see the graph) — the engine enforces it on the cell's behalf.
- **`CellValue`** — a number or an error. The closed set of things a cell can be. Immutable.
- **`Term`** — one signed element of a parsed expression: a ref, a literal, or (after the range follow-up) a nested aggregate.
- **`Parsed`** — the parser's return value. Deliberately not stored; it carries parse output from the parse phase to the mutate phase so nothing is parsed twice.

Direction that matters: `dependents` maps **precedent -> dependents** (`A1 -> {A3}`), because the write path asks *"who must I recompute?"*. The read path never walks the graph. `Cell.refs` is the reverse edge for one cell, and is what makes both edge teardown and in-degree counting O(1) per cell instead of a scan.

## 3. Class outlines

```
enum ValueKind: NUMBER | ERROR
enum ErrorCode: DIV_ZERO | REF        // "#DIV/0!", "#REF!"

class CellValue                        // immutable
- kind:   ValueKind
- number: double                       // when kind == NUMBER
- error:  ErrorCode                    // when kind == ERROR
+ static number(d: double) -> CellValue
+ static error(e: ErrorCode) -> CellValue
+ isError() -> bool
+ plus(other: CellValue, sign: int) -> CellValue    // error-absorbing add

enum TermKind: REF | NUM | AGG

class Term                             // immutable
- kind:  TermKind
- sign:  int                           // +1 | -1
- ref:   string                        // kind == REF
- num:   double                        // kind == NUM
- agg:   AggKind                       // kind == AGG  (SUM | MIN | MAX)
- args:  List<Term>                    // kind == AGG

class Parsed                           // returned, never stored
- isFormula: bool
- literal:   double
- terms:     List<Term>
- refs:      Set<string>

class Cell
- value:   CellValue                   // INVARIANT: always current
- formula: string|null                 // original text; display only
- terms:   List<Term>                  // empty for a literal
- refs:    Set<string>                 // distinct refs across terms
+ isFormula() -> bool
+ value() -> CellValue

class SpreadsheetEngine                // the orchestrator / facade
- cellMap:    Map<string, Cell>
- dependents: Map<string, Set<string>> // precedent -> dependents; keys may be never-set cells

+ setCell(ref: string, content: string) -> void
      // throws InvalidContentException, CircularReferenceException
+ readCell(ref: string) -> CellValue

- parseContent(content: string) -> Parsed        // throws InvalidContentException
- evaluateFormula(terms: List<Term>) -> CellValue
- currentValue(ref: string) -> CellValue         // never-set -> number(0)
- reachesAny(target: string, refs: Set<string>) -> bool
- attachEdges(ref: string, oldRefs: Set<string>, newRefs: Set<string>) -> void
- propagate(start: string) -> void
- normalizeRef(ref: string) -> string
- isInBounds(ref: string) -> bool
```

## 4. Core method implementations

```
function setCell(ref, content) -> void:
    ref = normalizeRef(ref)
    if not isInBounds(ref): throw InvalidContentException

    // --- 1. PARSE. May throw. Nothing mutated. ---
    p = parseContent(content)

    // --- 2. LEGALITY. May throw. Still nothing mutated. ---
    if ref in p.refs:                            throw CircularReferenceException
    if p.isFormula and reachesAny(ref, p.refs):  throw CircularReferenceException

    // --- 3. MUTATE. Cannot throw, so FR6 needs no rollback path. ---
    cell = cellMap.get(ref)
    if cell == null:
        cell = new Cell(value = CellValue.number(0), formula = null,
                        terms = [], refs = {})
        cellMap.put(ref, cell)

    attachEdges(ref, cell.refs, p.refs)          // old torn down before new attached

    cell.terms   = p.terms
    cell.refs    = p.refs
    cell.formula = p.isFormula ? content : null
    cell.value   = p.isFormula ? evaluateFormula(p.terms)
                               : CellValue.number(p.literal)

    propagate(ref)


function propagate(start) -> void:
    // 1. Transitive dependents. start is EXCLUDED — setCell already made it fresh,
    //    and re-evaluating it would be wrong: a literal cell has empty terms,
    //    so evaluateFormula would sum to 0 and silently zero the cell.
    dirty = {}; stack = [start]
    while stack not empty:
        cur = stack.pop()
        for d in dependents.get(cur, {}):
            if d not in dirty:
                dirty.add(d); stack.push(d)
    if dirty is empty: return

    // 2. In-degree counts ONLY precedents inside dirty. Precedents outside dirty
    //    are already fresh, so they impose no ordering constraint.
    indeg = {}
    for d in dirty:
        indeg[d] = count of r in cellMap[d].refs where r in dirty

    // 3. Kahn. Topological, not BFS level-order: in a diamond
    //    (C <- A, C <- B, B <- A) a level-order walk can evaluate C
    //    while B is still stale. Kahn cannot.
    queue = [d in dirty where indeg[d] == 0]
    done  = 0
    while queue not empty:
        cur = queue.popFront(); done += 1
        c = cellMap[cur]
        c.value = evaluateFormula(c.terms)
        for d in dependents.get(cur, {}):
            if d in dirty:
                indeg[d] -= 1
                if indeg[d] == 0: queue.pushBack(d)

    assert done == size(dirty)     // shortfall => a cycle escaped step 2 of setCell


function evaluateFormula(terms) -> CellValue:
    acc = CellValue.number(0)
    for t in terms:
        v = (t.kind == NUM) ? CellValue.number(t.num)
          : (t.kind == REF) ? currentValue(t.ref)
                            : aggregate(t.agg, t.args)
        if v.isError(): return CellValue.error(REF)   // FR13: error absorbs
        acc = acc.plus(v, t.sign)
    return acc


function reachesAny(target, refs) -> bool:
    // A cycle is created iff some new precedent already transitively depends
    // on target. Walk dependents FORWARD from target; if we reach any new ref,
    // that ref is downstream of target, and pointing target at it closes a loop.
    // Only target's incoming edges change during a set, so the live graph
    // answers this with no speculative mutation and no undo.
    seen = {target}; stack = [target]
    while stack not empty:
        cur = stack.pop()
        for d in dependents.get(cur, {}):
            if d in refs: return true
            if d not in seen: seen.add(d); stack.push(d)
    return false


function attachEdges(ref, oldRefs, newRefs) -> void:
    for r in oldRefs - newRefs:
        s = dependents[r]; s.remove(ref)
        if s is empty: dependents.remove(r)      // keep the map sparse
    for r in newRefs - oldRefs:
        dependents.getOrCreate(r, {}).add(ref)
```

**Edge cases covered:** empty content · non-numeric literal · out-of-bounds ref · consecutive operators · trailing operator · self-reference · direct 2-cycle · transitive n-cycle · formula replaced by literal (edges torn down) · reference to a never-set cell · diamond dependency (Kahn) · a cell whose only precedents are outside the dirty set · error absorbing through arbitrary depth.

**Worked trace** — `A1="1"`, `B1="=A1+1"`, `C1="=A1+B1"`, then `setCell("A1","10")`:

| Step | State |
|---|---|
| after 3 sets | `A1=1, B1=2, C1=3`; `dependents = {A1: {B1, C1}, B1: {C1}}` |
| `setCell("A1","10")` mutate | `A1.value = 10` (literal, written directly) |
| `propagate("A1")` step 1 | `dirty = {B1, C1}` — **A1 excluded** |
| step 2 | `indeg[B1] = 0` (its only ref A1 is not in dirty) · `indeg[C1] = 1` (B1 is in dirty; A1 is not) |
| step 3 | queue `[B1]` -> `B1 = A1+1 = 11`, `indeg[C1] -> 0`, push C1 -> `C1 = A1+B1 = 10+11 = 21` |
| `readCell("C1")` | **21** |

## 5. Design decisions

| Decision | Alternative | What it gives up |
|---|---|---|
| **Eager recompute on write, O(1) read** | Lazy: store formulas, compute on read with memo + dirty flags | Writes cost the size of the affected subtree. Chosen because FR7 makes reads the hot path, and lazy invalidation still has to walk the same subtree to mark dirty — so lazy pays the walk *and* adds staleness bookkeeping |
| **Kahn topological order in propagate** | BFS level order from the changed cell | Nothing. BFS is simply wrong on a diamond: it can evaluate a dependent before a sibling precedent is fresh. Same complexity, correct result |
| **Parse -> check -> mutate, mutate cannot throw** | Mutate optimistically, roll back on cycle | Requires carrying the parse result between phases. Buys FR6 structurally: there is no rollback path to get wrong, and no partial-write window |
| **Cycle check by walking `dependents` forward from target** | Insert the edges, run full cycle detection, remove on failure | Requires reasoning about why forward-from-target is sufficient (only target's in-edges change). Buys: never mutates speculatively, so FR6 holds even if detection throws |
| **`dependents` stores precedent -> dependents** | Store dependent -> precedents | Cheap in-degree counting would be lost. The write path always asks "who do I dirty?"; `Cell.refs` covers the reverse direction for the one cell that needs it |
| **`CellValue` as a closed number-or-error type** | `double` plus a NaN sentinel, or a parallel `error` field | A little ceremony. Buys the entire error follow-up for free and makes "what can a cell hold?" a compile-time question |
| **`Parsed` returned but never stored** | Store the raw string, re-parse on recompute | Nothing. Re-parsing on every propagate makes a deep chain O(n · formula length) for no benefit |
| **No design pattern at all** | Observer for recompute; Strategy for term kinds; Factory for `Cell` | Nothing — this is the right answer. `dependents` *is* the observer registry, and formalizing it as Observer hides the ordering guarantee the design depends on: Observer notifies in registration order, correctness here requires topological order. The pattern would actively cost you the thing that makes this work |

## 6. Extensibility — where each follow-up lands

- **Range functions (`=SUM(A1:A10)`)** — lands entirely in `parseContent`, which expands the range into individual refs and emits an `AGG` term. `dependents`, `propagate`, `reachesAny`, and `readCell` never learn ranges exist. The real design point: aggregation belongs to a **`Term`**, not to a `Cell` — `"=SUM(A1:A3)+MAX(B1:B2)-4"` has two different aggregates in one cell, so a cell-level `aggregationType` field cannot represent it.
- **Error values** — lands in `CellValue` plus one line of `evaluateFormula`. Errors propagate through `propagate` identically to numbers, because propagate only calls `evaluateFormula` and stores what it returns; it never inspects the value. That is the seam paying off.
- **Multiplication and division** — `Term.sign: int` becomes an operator with precedence, so `parseContent` grows a precedence climb and `evaluateFormula` walks a small expression tree instead of a flat list. Nothing outside those two methods changes.
- **Undo** — `setCell` already computes everything an undo record needs: the ref, the old `(value, formula, terms, refs)`, and the old edge set. Push that tuple on a stack in the mutate phase; undo replays `setCell` with the old content. No new structure.
- **Cross-sheet references** — the only assumption to break is that a ref key is unique within one map. Key by `(sheetId, ref)` and every algorithm above is untouched.

## 7. Concurrency

**Category: correctness.** Not coordination (no handoff) and not scarcity (no fixed pool). The failure is a check-then-act interleaving across two maps: T2's `reachesAny` reads a graph T1 is mid-mutation of, or T2's `evaluateFormula` reads `A1.value` before T1 writes it while T1's `propagate` computed `dirty` before T2's edge `A1 -> B1` existed. B1 ends at 1 and is never recomputed — a permanently stale value, with no error and no way to detect it.

**Smallest primitive that works: one lock on the engine, held across the whole of `setCell`.** Not per-phase, and not one lock per map: the invariant spans `cellMap` and `dependents` together, and the cycle check is only sound if no edge changes between check and mutate. Two finer locks reintroduce exactly the race they were meant to remove.

**Where it lives:** a single `ReadWriteLock` field on `SpreadsheetEngine` — write lock around the entire body of `setCell` (all three phases), read lock around `readCell`. Not on `Cell`: no rule spans a single cell, so a per-cell lock would guard nothing that matters.

**Cost.** Writes serialize completely — one writer at a time, even for sets touching disjoint regions of the sheet. Reads stay concurrent with each other and remain O(1), the right trade when reads dominate. A long `propagate` over a large dirty set blocks every other writer for its duration.

**If write throughput became the bottleneck**, the next step is *not* per-cell locks — it is partitioning the sheet into connected components of the dependency graph, one lock each, since a set can only dirty cells within its own component. That preserves the span of the invariant, which per-cell locking does not: per-cell locking would require acquiring every lock in the dirty set in a global order to avoid deadlock, and the dirty set is not known until after the graph has been read. That chicken-and-egg is what makes it the wrong tool here.
