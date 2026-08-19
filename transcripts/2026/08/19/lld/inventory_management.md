# LLD Round Transcript
**Date:** 2026-08-19
**Start Time:** 09:11:43 · **End Time:** 09:59:03 · **Duration:** 47 min
**Problem:** Inventory Management System (multi-warehouse)
**Category:** product domain
**Difficulty:** 3/5 — one real rule to place (per-(warehouse, SKU) capacity), one genuine trade-off (atomic multi-line fulfilment across warehouses), object model fits comfortably in 40 min
**Performance Rating:** 3/5  <!-- machine-read on future rounds; <=3 = eligible for re-ask, >=4 retired -->
**Hints Used:** 0/2
**Requirements Asked:** item type · actors and system function · rules for adding stock and fulfilling orders · whether orders are managed here · single vs. multiple warehouses · whether a SKU line can be split across warehouses
**Never Asked:** lifecycle / non-order stock removal · failure convention (adopted unilaterally at class-design time) · concurrency posture · who seeds the capacity map · reservations/holds
**Walk coverage:** 5/8 hit (1 partial, 2 miss)
**Out of Scope list produced:** Unprompted (thin — 2 items)
**Self-Verified:** No — trace only when asked; his claimed trace was **correct**
**Concurrency follow-up:** Correctness / check-then-act (two clients race the last 3 units of SKU-A). Called it a "reliability issue" rather than naming the category; picked a coarse lock, stated the throughput cost unprompted; when pushed to (warehouse, SKU) granularity, immediately named deadlock and total lock ordering
**Would it have fit a real 40-min round?** No — cut off during the extensibility follow-ups; the concurrency curveball and the damage/shrinkage follow-up (his best material) would never have happened

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements + Out of Scope | 5 min | 11.3 min | +6.3 | No |
| Entities & relationships | 8 min | 16.3 min | +8.3 | No |
| Class design | 20 min | 29.7 min | +9.7 | No |
| Implementation + trace | 32 min | 34.0 min (trace at 45.1) | +2.0 | Borderline |
| Extensibility + concurrency | 40 min | 47.3 min | +7.3 | No |
| **Total** | 40 min | **47.3 min** | **+7.3** | No |

---

## Problem Statement

**Design an inventory management system for a warehouse.**

**Difficulty: 3/5**

Five phases: requirements + Out of Scope (5 min) · entities & relationships (8) · class design (20) · implementation + trace (32) · extensibility + concurrency (40). Clock measured, not enforced. No UML. Language: pseudo-code.

---

## Conversation Log

**Interviewer [09:11:43]:** Design an inventory management system for a warehouse. Difficulty 3/5. Which language for phase 4? Any questions before you start?

**Aayush:** 1. What kind of items will be in the inventory? 2. Who will interact with the inventory management system and what function will the inventory management system serve?

**Interviewer [09:13:15]:** Physical goods identified by SKU. Each SKU is a distinct product; units of the same SKU are interchangeable, no serial numbers. Two programmatic clients of a single in-process API: a warehouse operations client (receives shipments, fulfils outbound orders) and a store/ordering client (asks about availability, places demands). No UI.

**Aayush:** Are there any rules for adding inventory and for fulfilling orders?

**Interviewer [09:16:39]:** Adding stock — SKU must already be registered in the catalogue, unknown SKU illegal; quantity must be positive; each SKU has a maximum capacity in the warehouse and exceeding it is illegal. Fulfilling an order — an order is a set of (SKU, quantity) lines, potentially several; all-or-nothing; can't fulfil more than available.

**Aayush:** Do we manage the orders as well?

**Interviewer [09:19:00]:** No. Order lifecycle, payment, shipping, customer records are someone else's system. An order arrives as an already-formed request; you either fulfil it against stock or reject it. What you own is the stock.

**Aayush:** [requirements list — see His Design]

**Interviewer [09:21:14]:** Recorded exactly as given. Which language for phase 4?

**Aayush:** Pseudocode. And does the system need to support multiple warehouses or just one?

**Interviewer [09:22:23]:** Multiple warehouses. Each holds its own stock and has its own per-SKU capacity limit — capacity is per (warehouse, SKU), not global. A receiving call names the warehouse. An outbound order does not name a warehouse; the system decides where stock comes from.

**Aayush:** Update the requirement with the new support for multiple warehouses.

**Interviewer [09:23:01]:** Added as requirement 4. Phase 2 — entities and relationships.

**Aayush:** Entities: 1. Warehouse (name, Map<ProductSKU,int> inventoryMap) 2. enum ProductSKU 3. InventoryManagementSystem(Map<string,Warehouse>)

**Interviewer [09:25:18]:** (1) Requirement 3 says each warehouse has a capacity limit per SKU and exceeding it is illegal. Nothing in your entity list holds that number. Where does it live, and who checks it? (2) You made ProductSKU an enum. What does that buy over a plain string id, and what does it cost? (3) Which of these three is the orchestrator?

**Aayush:** There should also be Order(productSkuDesired: ProductSKU and quantity).

**Interviewer [09:25:49]:** Recorded. My three questions are still open.

**Aayush:** 1. Each warehouse will have its own Map for productSKU to capacity limit. 2. ProductSKU being an enum provides a centralised place for all the SKU and adding new SKU becomes easy — we just need to update the enum in one place. With a simple string there could be typos in different places. 3. The InventoryManagementSystem is the point of contact exposing methods to perform the various operations required by this system.

**Interviewer [09:27:59]:** (1) The business onboards 500 new SKUs a week, entered by a merchandising team through an admin tool. Walk me through what happens with your enum. (2) You told me where the capacity map lives, not who checks it. When a receiving call arrives for 40 units of SKU-1234 at warehouse "BLR-1", which object decides whether that's legal? Then go to phase 3.

**Aayush:** We can use a hashset or a persistent database to store SKUs. The warehouse should verify the capacity requirement. [class outlines — revision 1, all-bool returns]

**Interviewer [09:28:17]:** (1) Your signatures still say ProductSKU (the enum) but you just said hashset or database — which is it? (2) `getInventory(sku): int` on the orchestrator, no warehouse argument — what number is that? (3) Order has two getters and nothing else — who calls getQuantity(), and what do they do with the value? (4) Everything returns bool. When placeOrder returns false, what does the client know?

**Aayush:** [class outlines — revision 2: throw convention, named exception types, void mutators, `fulfillOrder(order: Order) -> Order`, Order gains `fulfillingWarehouse: Warehouse | null`]

**Interviewer [09:38:26]:** Failure convention is now throw — I'll hold you to that. Still open: enum vs. hashset/database; what number getInventory returns; and here's an order from the store client — 2 units of SKU-1234, 5 units of SKU-9876, 1 unit of SKU-5555, one order, three lines, all-or-nothing. Show me the call the client makes. Also: who calls getSKU(), getQuantity(), setFulfillingWarehouse(), and what decision are they making?

**Aayush:** Update the md file as well.

**Interviewer [09:39:02]:** Already in the file — "Class outlines — revision 2". Four questions still open.

**Aayush:** 1. Going with enum. 2. It can be used to get the total available inventory for a SKU across all warehouses managed by the system. 3. Order needs to have a map of SKU to quantity to capture multiple SKUs in the order. 4. The InventoryManagementSystem calls the methods of the Order class to obtain information about the order, in order to decide whether the order can be fulfilled or not.

**Interviewer [09:41:22]:** Recorded. Phase 4 — write pseudo-code for `InventoryManagementSystem.fulfillOrder(order)` and `Warehouse.addInventory(sku, quantity)`. Keep the throw convention.

**Aayush:** Is it necessary for one SKU to be filled from one warehouse, or can an order be fulfilled using multiple warehouses?

**Interviewer [09:43:34]:** An order may draw from multiple warehouses, and a single SKU line may itself be split across warehouses — if BLR-1 has 3 units and BLR-2 has 2, a line for 5 units is satisfiable. All-or-nothing still holds across the whole order.

**Aayush:** [implementation — see His Design]

**Interviewer [09:48:02]:** (1) Walk me through fulfillOrder with: order 2×A, 5×B, 1×C; BLR-1 holds A:3 B:2; BLR-2 holds A:0 B:4; neither holds C. Step by step, and what inventoryMap looks like when the call returns. (2) Product says the warehouse chosen must now be the one nearest the customer's shipping address — and next quarter it'll be "oldest stock first". Where does that change land? (3) Two order clients call fulfillOrder at the same time, both wanting the last 3 units of SKU-A. What breaks, and what do you do?

**Aayush:** 1. The order can't be fulfilled because no warehouse has any C. So A would be fulfilled from BLR-1 and B from BLR-1 (2) BLR-2 (3), but when trying to fill C no warehouse has it, so exception thrown. The inventory maps of the warehouses are updated only after verifying that the order can be fulfilled, so no changes to inventory map. 2. We would need to have a strategy for fulfilling orders. This strategy would take as input the order to be fulfilled and the warehouses from which the order can be fulfilled. Internally the strategy can get whatever data it needs by calling relevant methods of warehouse and order and determine how the order needs to be fulfilled. The strategy will just determine how the order should be fulfilled; the actual reduction of inventory will be separate from the strategy. 3. Both orders could see the warehouse having inventory and both could be fulfilled — this is a reliability issue. To prevent this we would need to make order fulfillment atomic. A simple solution is a lock to serialise the order fulfillment logic, but this will heavily reduce order fulfillment throughput since multiple orders for unrelated SKU items could wait while an order is being fulfilled.

**Interviewer [09:56:51]:** (4) New requirement: an ops client says "12 units of SKU-A at BLR-1 were destroyed in a flood." Where does that operation live, and what rules must it obey? (5) On your lock: you said serialising all fulfilment kills throughput for unrelated SKUs. So make it finer — what exactly do you lock, and what new failure mode does that introduce?

**Aayush:** The removal of the destroyed items would happen at the warehouse level. Since the destroyed items are not part of an order, the reduction of the inventory would need to be done by the warehouse, and there should be a public method exposed by the warehouse to remove such items. Removal of item should not make the inventory count negative and the quantity should be a valid quantity, that is a positive integer. 5. We could take a lock at (warehouse, SKU) level. This introduces a possibility of deadlock, since an order can be fulfilled from multiple warehouses and if multiple orders try to take locks in different orders it can lead to a deadlock, so we would need to have a consistent order of taking locks.

**Interviewer [09:59:03]:** Round over.

---

## Walk coverage (phase 1)
| # | Item | Hit/Partial/Miss | Evidence |
|---|---|---|---|
| 1 | Actors & entry point | Hit | "Who will interact with the inventory management system and what function will the inventory management system serve?" |
| 2 | Core operations | Hit | "add SKU items to warehouse inventory, match outbound orders with inventory items, report quantity of available SKU items" |
| 3 | Rules & legality | Hit | "are there any rules for adding inventory and for fulfilling orders?" — then wrote all of them down |
| 4 | Lifecycle & terminal states | Miss | Never asked whether stock leaves for any reason other than an order; damage/shrinkage only surfaced as an interviewer follow-up |
| 5 | Failure behaviour | Partial | Not asked in phase 1; adopted `throw` at class-design revision 2 and then held it on every signature |
| 6 | Multiplicity & domain variants | Hit | "does the system need to support multiple warehouse or just one?" and "is it necessary for one SKU to be filled from one warehouse?" |
| 7 | Concurrency posture | Miss | Never raised; only addressed once the curveball fired |
| 8 | Explicit Out of Scope | Hit | Produced unprompted: UI, order management |

**Dangling rules:**
- *"SKU must already be in catalog, unknown SKU illegal"* — his own stated requirement, enforced nowhere in the final design. No `Catalogue` object exists; the enum was treated as the catalogue while "adding new SKU to catalogue" was simultaneously declared out of scope. **This is the binding ceiling.**
- *`capacityMap` ownership* — declared on `Warehouse`, checked by `Warehouse`, but nothing ever populates it. With `getOrDefault(sku, 0)`, a freshly constructed warehouse rejects every receipt and blames capacity rather than construction. "We don't stock that here" and "we're full" are conflated into one exception.

**Silent assumptions:**
- *An order has one SKU line.* First `Order` was `(productSkuDesired, quantity)` and `fulfillOrder` took one — despite being told at +5m that an order is a set of lines, all-or-nothing. Fixed only when handed a concrete three-line order. Cost: the atomicity requirement, the most interesting rule in the problem, was absent from the design until minute 30.
- *The enum answers the catalogue-scale question.* Said "hashset or a persistent database" when pushed, then returned to the enum in his signatures. Cost: identified the right answer and didn't apply it.

---

## His Design

**Requirements he gathered:**
1. Inventory has physical goods identified by SKU. There can be multiple items with the same SKU in inventory (only quantity per SKU tracked).
2. Actions — add SKU items to warehouse inventory; match outbound orders with inventory items; report quantity of available SKU items.
3. Rules — Adding stock: SKU must already be in catalog, unknown SKU illegal. Quantity +ve. A warehouse has a capacity limit for each SKU; pushing past capacity is illegal. Fulfilling order: orders are fulfilled atomically.
4. (added after asking) Multiple warehouses. Each holds its own stock and has its own per-(warehouse, SKU) capacity limit. A receiving call names the warehouse; an outbound order does not.

**Out of Scope:**
1. UI
2. Order management

**Entities & relationships:**
- `Warehouse` (name, inventoryMap, plus its own capacity map) · `enum ProductSKU` · `InventoryManagementSystem` (warehouseMap) — the orchestrator / point of contact · `Order` (added second)

**Class outlines (final):**
```
class Warehouse:
    - name: String
    - inventoryMap: Map<ProductSKU, int>
    - capacityMap: Map<ProductSKU, int>
    + addInventory(sku: ProductSKU, quantity: int) -> void   // throws InvalidQuantity, ExceedingCapacity
    + getInventory(sku: ProductSKU) -> int
    + removeInventory(sku: ProductSKU, qty: int) -> void     // throws InvalidQuantity, ExceedingCurrentCount

class InventoryManagementSystem:
    - warehouseMap: Map<string, Warehouse>
    + addInventory(sku: ProductSKU, quantity: int, warehouseName: string) -> void  // throws InvalidQuantity, ExceedingCapacity
    + getInventory(sku: ProductSKU) -> int                   // total across all warehouses
    + fulfillOrder(order: Order) -> Order                    // throws if order can't be fulfilled from any warehouse

class Order:
    - sku: ProductSKU        // later: a Map of SKU -> quantity
    - quantity: int
    - fulfillingWarehouse: Warehouse | null
    + getSKU() -> ProductSKU
    + getQuantity() -> int
    + setFulfillingWarehouse(warehouse: Warehouse) -> void
```

**Core implementation:**
```
fullfillOrder(order:Order) -> Order:
   itemsMap = order.getItemsMap()
   fulfillingWarehousesMap = Map<ProductSKU,Map<string,int>>
   for sku,quantity in itemsMap:
      remaining = quantity
      warehouseCount = Map<string,int>
      for name,warehouse in warehouseMap:
          available = warehouse.getInventory(sku)
          if available == 0: continue
          take = min(available,remaining)
          warehouseCount[name] = take
          remaining -= take
          if remaining == 0: break
      if remaining > 0:
          throw OrderCannotBeFulfilled
      fulfillingWarehousesMap[sku] = warehouseCount

   for sku, warehouseCount in fulfillingWarehousesMap:
       for name,qty in warehouseCount:
           warehouseMap[name].removeInventory(sku,qty)
   order.setFulfillingWarehouses(fulfillingWarehousesMap)
   return order

addInventory(sku:ProductSKU,quantity:int) -> void:
   if quantity <= 0:
      throw InvalidQuantity
   currentQuantity = inventoryMap.getOrDefault(sku,0)
   capacity = capacityMap.getOrDefault(sku,0)
   if currentQuantity + quantity > capacity:
      throw ExceedingCapacity
   inventoryMap[sku] = currentQuantity + quantity
```

**Gaps / misplaced responsibilities:**
- No `Catalogue`. The stated "unknown SKU is illegal" rule has no owner and no code.
- `capacityMap` never populated — no constructor. As written, `addInventory` throws `ExceedingCapacity` on every call to a fresh warehouse, and blames the wrong thing.
- `addInventory` collapses "warehouse doesn't stock this SKU" and "warehouse is full" into one exception.
- `InventoryManagementSystem.addInventory` never implemented; unknown `warehouseName` has a declared throw contract with no code behind it.
- `Order` is a data class with getters; the service assembles demand from them and makes the decision — Ask, not Tell, stated explicitly as design intent.
- `Order.fulfillingWarehouse` mutates the outcome onto the request object; `fulfillingWarehousesMap` is an `AllocationPlan` living as a local variable instead of a type.
- Signature drift: outline declares `setFulfillingWarehouse(warehouse: Warehouse)`, code calls `setFulfillingWarehouses(map)`.
- Duplicate-SKU coalescing never considered (masked because `Order` uses a Map).

**What was right:** the plan-then-commit split in `fulfillOrder` is the correct shape and delivers atomicity with no rollback path. The capacity rule is correctly homed on `Warehouse`. Zero gratuitous patterns. The Strategy answer separated deciding the allocation from applying it, unprompted.

---

## Feedback Given

### Round conditions
- **Hints used: 0/2.** No ceiling from hints.
- **Asked:** item type · actors and system function · rules for adding stock and fulfilling orders · whether orders are managed here · single vs. multiple warehouses · whether a SKU line can be split across warehouses.
- **Never asked:** lifecycle / non-order stock removal · failure convention · concurrency posture · who seeds the capacity map · reservations/holds.
- **Out of Scope: produced unprompted** (thin — two items). Breaks a five-session streak.
- **Self-verification: prompted, not volunteered.** When asked, his trace was **correct** — identified that C fails, that the exception fires, and that no `inventoryMap` is touched because commit runs only after planning succeeds.

### Rubric
| Dimension | Score | Reason |
|---|---|---|
| Requirements & scoping | 4/5 | 5 hits, unprompted Out of Scope, and the split-across-warehouses question is a genuinely senior probe. Held back by never asking lifecycle or concurrency, and by leaving the unknown-SKU rule unowned |
| Entity modelling | 3/5 | Right three entities up front, orchestrator correctly identified. But `Order` modelled as a single line despite a stated multi-line requirement, no `Catalogue` for a rule he stated, no object for the allocation outcome |
| Class design | 3/5 | Revision 2 is clean: consistent throw convention, named exception per rule, void mutators. Revision 1 was all-bool with no discrimination. `capacityMap` has no constructor; signature drift on `setFulfillingWarehouse(s)` |
| Responsibility placement | 3/5 | Capacity rule correctly homed on `Warehouse`, stated unprompted when asked who checks. Against that: `Order` is a pure data class and he explicitly described the service reading its getters to make the decision; outcome mutated onto the request |
| Implementation & correctness | 3/5 | `fulfillOrder` genuinely correct — plan-then-commit gives atomicity with no rollback path. Offset by `addInventory` collapsing two failures, `IMS.addInventory` never written despite a declared throw contract, and no trace until asked |
| Simplicity & judgement | 4/5 | Zero gratuitous patterns, no speculative fields. The one pattern reached for (Strategy, under pressure) is the one that belongs. The enum is the lone judgement error, and a real one — it moves the catalogue from data into code |
| Extensibility | 4/5 | Both follow-ups landed on seams. Separated deciding the allocation from applying it, unprompted. Damage/shrinkage correctly homed on `Warehouse` with both invariants named |
| Concurrency | 3/5 | Called it a "reliability issue" — it is **correctness / check-then-act**, and naming the category is half the answer. But picked the smallest primitive first, stated its cost unprompted, and on being pushed to (warehouse, SKU) locks immediately named deadlock and total lock ordering |
| Communication | 3/5 | Clear and responsive, fixed things fast under pushback. But left four direct questions unanswered across two turns requiring two re-asks, and phases 1–3 consumed 30 of 40 minutes |

### Performance Rating: 3/5
Would have been a 4 — zero hints, best walk coverage to date, correct core logic, both follow-ups on seams — **capped at 3**: he raised "unknown SKU is illegal" as his own requirement and the final design enforces it nowhere.

### Pace report
47.3 min against a 40-min box on a difficulty-3 problem. **Would not have fit.** A real interviewer cuts him off during the extensibility follow-ups — he gets the Strategy question and nothing after it, meaning the concurrency curveball never fires, and concurrency is the most-weighted senior signal for this problem class. Everything after minute 40 — the damage/shrinkage answer and the whole lock-granularity discussion, his best material — would have been invisible.

Biggest sink: the 10 minutes between the entity list and the first class outline (09:28 → 09:38), with four questions open the whole time, producing a revision that changed the failure convention wholesale — rework that belonged in phase 1, item 5 of the walk. Second sink: phases 1 and 2 took 16 minutes, cost driven by *serialising* the questions one per turn rather than by the questions themselves. **The overrun was process, not problem size** — a 3 should finish inside the box with room to spare.

### Senior-signal scorecard
| Signal | Status | Reason |
|---|---|---|
| Scopes before designing | Strong | Asked before designing, produced Out of Scope unprompted, asked the multiplicity question he had skipped every previous round |
| State derived from requirements | Mixed | `capacityMap` traced to a stated rule; but `Order.fulfillingWarehouse` records an outcome on a request object, and the SKU catalogue rule got no state at all |
| Rules live with their state (Tell, Don't Ask) | Mixed | Right instinct on `Warehouse`, twice, unprompted. Wrong on `Order`, and he articulated the Ask pattern explicitly as design intent |
| Simplicity held under pressure | Strong | Multi-warehouse, splitting, damage, concurrency — nothing bloated, zero unjustified patterns |
| Verifies own logic | Mixed | Did not trace unprompted; traced correctly when asked — first correct prompted trace, and it caught the property that matters |
| Extends without rewriting | Strong | Both follow-ups absorbed at a seam; the plan/apply separation was unprompted design insight |

**Overall: mid-level with real senior edges — hire, not strong-hire.** The gap is not modelling ability; it is that stated rules don't reliably become enforced code, and 30 of 40 minutes go to talking about the design rather than writing it.

### What a senior strong-hire would have done on this problem
- **Skipped requirement question:** "can stock leave inventory for reasons other than an order?" — surfaces damage, audit adjustments and inter-warehouse transfers, all wanting the same `Warehouse.adjust(sku, delta)`. Asked at minute 3 it's scoping; answered at minute 50 it's rescue.
- **The rule with no owner:** "unknown SKU is illegal" needs a `Catalogue` — one class, one `Set[str]`, one `require(sku)` that throws `UnknownSku`. It also dissolves the enum problem: the typo-safety wanted from the enum comes from `require()` without freezing the catalogue into compiled code. He *said* "hashset or database" and then didn't do it.
- **The getter that should have been a behavior:** `order.demand() -> Dict[Sku, int]`, coalescing duplicate lines, instead of the service reading `getSKU()`/`getQuantity()` and computing demand itself. The coalescing matters — two lines for the same SKU planned independently double-book the same units.
- **The field that shouldn't exist:** `Order.fulfillingWarehouse`. An `AllocationPlan` return costs one class and buys a result you can build, validate and throw away. He already built it — `fulfillingWarehousesMap` is exactly that object, living as a local variable instead of a type.
- **The pattern that genuinely pays:** Strategy for allocation ordering — found correctly, under pressure, because the requirement forced it. Right way round.
- **The edge case a trace would have caught:** `addInventory("SKU-A", 10)` on a freshly constructed `Warehouse` — `getOrDefault(sku, 0)` returns 0, `0 + 10 > 0`, `ExceedingCapacity`. Nothing populates that map, so the warehouse cannot accept a single unit of anything, and the error blames capacity rather than construction.

### Drill
Take today's design. Write the constructors for `Warehouse` and `InventoryManagementSystem`, then write the **six** exceptions the two classes can throw, each with the exact condition that raises it. Timebox 10 minutes. Every round leaves one stated rule without code behind it, and every round it's a rule whose state has no constructor seeding it. Constructors first, then rules.

Then the pre-round checklist in `lld_senior_guidance.md` — items 3 ("every rule raised got a resolution") and 10 ("traced one concrete scenario out loud").
