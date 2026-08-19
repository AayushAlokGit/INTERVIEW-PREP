# Inventory Management System — Design Sheet

## Phase 1 — Requirements

1. Inventory has physical goods identified by SKU. There can be multiple items with the same SKU in inventory (only quantity per SKU tracked).
2. **Actions** — add SKU items to warehouse inventory; match outbound orders with inventory items; report quantity of available SKU items.
3. **Rules**
   - Adding stock: SKU must already be in catalog, unknown SKU illegal. Quantity must be +ve. A warehouse has a capacity limit for each SKU; pushing past capacity is illegal.
   - Fulfilling order: orders are fulfilled atomically.
4. Multiple warehouses are supported. Each warehouse holds its own stock and has its own per-SKU capacity limit � capacity is per (warehouse, SKU), not global. A receiving call names the warehouse the goods arrive at. An outbound order does not name a warehouse; the system decides where the stock comes from.

## Out of Scope

1. UI
2. Order management

## Phase 2 — Entities

1. `Warehouse` — `name`, `Map<ProductSKU, int> inventoryMap`
2. `enum ProductSKU`
3. `InventoryManagementSystem` — `Map<string, Warehouse>`
4. `Order` — `productSkuDesired: ProductSKU`, `quantity`

**Relationships / clarifications given:**
- Each `Warehouse` also holds its own `Map<ProductSKU, capacityLimit>`.
- `ProductSKU` is an enum: centralised place for all SKUs, adding a new SKU means updating the enum in one place, avoids typos that plain strings would allow.
- `InventoryManagementSystem` is the orchestrator / point of contact, exposing methods for the system's operations.

## Phase 3 — Class outlines

SKU storage: "we can use a hashset or a persistent database to store SKUs."
Capacity check: "the warehouse should verify the capacity requirement."

### Warehouse
```
- name: string
- inventoryMap: Map<ProductSKU, int>
- capacityMap: Map<ProductSKU, int>

+ addInventory(sku: ProductSKU, quantity: int): bool
+ getInventory(sku: ProductSKU): int
+ removeInventory(sku: ProductSKU, quantity: int): bool
```

### InventoryManagementSystem
```
- warehouseMap: Map<string, Warehouse>

+ addInventory(sku: ProductSKU, quantity: int, warehouseName: string): bool
+ getInventory(sku: ProductSKU): int
+ placeOrder(orders: List<Order>): bool
```

### Order
```
- productSku: ProductSKU
- quantity: int

+ getProductSku(): ProductSKU
+ getQuantity(): int
```

### Class outlines — revision 2

```
class Warehouse:
    - name: String
    - inventoryMap: Map<ProductSKU, int>
    - capacityMap: Map<ProductSKU, int>

    + addInventory(sku: ProductSKU, quantity: int) -> void
        // throws: InvalidQuantity, ExceedingCapacity
    + getInventory(sku: ProductSKU) -> int
    + removeInventory(sku: ProductSKU, qty: int) -> void
        // throws: InvalidQuantity, ExceedingCurrentCount

class InventoryManagementSystem:
    - warehouseMap: Map<string, Warehouse>

    + addInventory(sku: ProductSKU, quantity: int, warehouseName: string) -> void
        // throws: InvalidQuantity, ExceedingCapacity
    + getInventory(sku: ProductSKU) -> int
    + fulfillOrder(order: Order) -> Order
        // throws exception if order can't be fulfilled from any warehouse

class Order:
    - sku: ProductSKU
    - quantity: int
    - fulfillingWarehouse: Warehouse | null

    + getSKU() -> ProductSKU
    + getQuantity() -> int
    + setFulfillingWarehouse(warehouse: Warehouse) -> void
```

**Answers to probing questions:**
1. SKU representation: going with the enum.
2. `InventoryManagementSystem.getInventory(sku)` returns the total available inventory for a SKU across all warehouses managed by the system.
3. `Order` needs to have a map of SKU to quantity to capture multiple SKUs in one order.
4. The `InventoryManagementSystem` calls the methods of the `Order` class to obtain information about the order, in order to decide whether the order can be fulfilled or not.

## Phase 4 — Core implementation (pseudo-code)

```
fulfillOrder(order: Order) -> Order:
   itemsMap = order.getItemsMap()
   fulfillingWarehousesMap = Map<ProductSKU, Map<string, int>>
   for sku, quantity in itemsMap:
      remaining = quantity
      warehouseCount = Map<string, int>
      for name, warehouse in warehouseMap:
          available = warehouse.getInventory(sku)
          if available == 0: continue
          take = min(available, remaining)
          warehouseCount[name] = take
          remaining -= take
          if remaining == 0: break
      if remaining > 0:
          throw OrderCannotBeFulfilled
      fulfillingWarehousesMap[sku] = warehouseCount

   for sku, warehouseCount in fulfillingWarehousesMap:
       for name, qty in warehouseCount:
           warehouseMap[name].removeInventory(sku, qty)
   order.setFulfillingWarehouses(fulfillingWarehousesMap)
   return order


addInventory(sku: ProductSKU, quantity: int) -> void:
   if quantity <= 0:
      throw InvalidQuantity
   currentQuantity = inventoryMap.getOrDefault(sku, 0)
   capacity = capacityMap.getOrDefault(sku, 0)
   if currentQuantity + quantity > capacity:
      throw ExceedingCapacity
   inventoryMap[sku] = currentQuantity + quantity
```

## Phase 5 — Extensibility + concurrency follow-ups

Not reached — the candidate stepped away after submitting the implementation. No self-verification trace was performed, and the concurrency curveball never fired.

---
---

# Optimal Reference (what a senior strong-hire would design)

## 1. Requirements + Out of Scope

**Functional**
- **FR1** — Stock is tracked per `(warehouse, SKU)` as a quantity. Units of a SKU are fungible.
- **FR2** — `receive(warehouseId, sku, qty)`: adds stock. Legal iff SKU is in the catalogue, `qty > 0`, and `onHand + qty <= capacity(warehouse, sku)`.
- **FR3** — `availability(sku)` reports units available across all warehouses; `availabilityAt(warehouseId, sku)` reports it for one.
- **FR4** — `fulfill(order)`: an order is a set of `(sku, qty)` lines. Atomic across the whole order. A line may be split across warehouses.
- **FR5** — Capacity is per `(warehouse, SKU)`. A warehouse with no capacity entry for a SKU cannot stock it — and that is a *distinct* rejection from "capacity exceeded".
- **FR6** — Failure convention: **throw**, one exception type per rule (`UnknownSku`, `UnknownWarehouse`, `InvalidQuantity`, `CapacityExceeded`, `InsufficientStock`). Held on every signature.
- **FR7** — The result of a fulfilment is an **allocation plan** — which warehouse gave how many of which SKU — returned to the caller, not mutated onto the request object.

**Questions that were never asked, and what each would have changed**

| Question | Answer | Design impact |
|---|---|---|
| Are reservations/holds needed before fulfilment? | No — fulfil deducts immediately | Removes a whole state machine; makes single-phase commit legal |
| Can stock leave for non-order reasons (damage, audit, transfer)? | Yes — shrinkage adjustments exist | `adjust(warehouseId, sku, delta)` becomes a first-class operation, not a private helper |
| Is there a preferred warehouse ordering (nearest, fullest, oldest stock)? | Yes — pluggable | This is the **one genuine Strategy seam** in the problem |
| Is partial fulfilment ever acceptable? | No | Confirms the two-phase plan/commit shape |
| Is the object shared across threads? | Yes in production; single-threaded for this round | Decides *now* where the lock lives |
| Who seeds the capacity map? | Warehouse construction, from config | Without this the design cannot accept a single unit of stock |

**Out of Scope:** order lifecycle / payment / shipping / customers · catalogue administration (SKU registration) · persistence and durability · authn/authz · item-level serial tracking · bin/shelf location inside a warehouse · pricing.

## 2. Entities & relationships

- **`InventoryService`** — *the orchestrator*. Owns the warehouse registry, the catalogue and the allocation strategy. Owns exactly the policy that is cross-warehouse: allocation ordering and order atomicity. Nothing else is public.
- **`Warehouse`** — owns its own stock and its own capacity, and enforces every rule decidable from *its own* numbers: positive quantity, capacity ceiling, sufficient stock, is-this-SKU-stockable-here.
- **`Catalogue`** — owns SKU registration and the single answer to "is this SKU real?".
- **`Order`** — an immutable request: a set of lines. It answers questions about itself; it is never mutated with the outcome.
- **`AllocationPlan`** — the outcome: `warehouseId -> sku -> qty`. This is the object that makes atomicity expressible.
- **`AllocationStrategy`** (interface) — decides the order in which warehouses are consulted for a SKU.

```
InventoryService --owns--> Warehouse*, Catalogue, AllocationStrategy
InventoryService.fulfill(Order) -> AllocationPlan
```

## 3. Class outlines

```
class Sku:                                   # value object, NOT an enum
    - id: str                                # the catalogue is data, not code
    + __eq__ / __hash__

class Catalogue:
    - skus: Set[str]
    + contains(sku: Sku) -> bool
    + require(sku: Sku) -> None              # throws UnknownSku

class Warehouse:
    - id: str
    - onHand: Dict[Sku, int]
    - capacity: Dict[Sku, int]               # seeded at construction

    + __init__(id: str, capacity: Dict[Sku, int])
    + availableOf(sku: Sku) -> int
    + canAccept(sku: Sku, qty: int) -> bool
    + receive(sku: Sku, qty: int) -> None     # throws InvalidQuantity, SkuNotStockedHere, CapacityExceeded
    + issue(sku: Sku, qty: int) -> None       # throws InvalidQuantity, InsufficientStock
    + adjust(sku: Sku, delta: int) -> None    # throws CapacityExceeded, InsufficientStock

class OrderLine:
    - sku: Sku
    - qty: int

class Order:
    - id: str
    - lines: List[OrderLine]                  # immutable
    + lines() -> Iterable[OrderLine]

class AllocationPlan:
    - takes: Dict[str, Dict[Sku, int]]        # warehouseId -> sku -> qty
    + add(warehouseId: str, sku: Sku, qty: int) -> None
    + takesFor(warehouseId: str) -> Dict[Sku, int]
    + entries() -> Iterable[(str, Sku, int)]

class AllocationStrategy(interface):
    + order(warehouses: List[Warehouse], sku: Sku, qty: int) -> List[Warehouse]

class FewestWarehousesFirst(AllocationStrategy): ...
class NearestToCustomer(AllocationStrategy): ...

class InventoryService:
    - warehouses: Dict[str, Warehouse]
    - catalogue: Catalogue
    - strategy: AllocationStrategy

    + receive(warehouseId: str, sku: Sku, qty: int) -> None
        # throws UnknownWarehouse, UnknownSku, InvalidQuantity, SkuNotStockedHere, CapacityExceeded
    + availability(sku: Sku) -> int
    + availabilityAt(warehouseId: str, sku: Sku) -> int
    + fulfill(order: Order) -> AllocationPlan
        # throws UnknownSku, InvalidQuantity, InsufficientStock
```

## 4. Core method implementations

```python
# --- InventoryService ---

def receive(self, warehouse_id: str, sku: Sku, qty: int) -> None:
    self.catalogue.require(sku)               # unknown SKU is its OWN failure, not a capacity failure
    wh = self.warehouses.get(warehouse_id)
    if wh is None:
        raise UnknownWarehouse(warehouse_id)
    wh.receive(sku, qty)                      # Tell, don't Ask — no availableOf() read here


# --- Warehouse: every rule decidable from its own numbers lives here ---

def receive(self, sku: Sku, qty: int) -> None:
    if qty <= 0:
        raise InvalidQuantity(qty)
    if sku not in self.capacity:              # "we don't carry that here" != "we're full"
        raise SkuNotStockedHere(self.id, sku)
    on_hand = self.on_hand.get(sku, 0)
    if on_hand + qty > self.capacity[sku]:
        raise CapacityExceeded(self.id, sku, on_hand, self.capacity[sku], qty)
    self.on_hand[sku] = on_hand + qty


def issue(self, sku: Sku, qty: int) -> None:
    if qty <= 0:
        raise InvalidQuantity(qty)
    on_hand = self.on_hand.get(sku, 0)
    if on_hand < qty:
        raise InsufficientStock(self.id, sku, on_hand, qty)
    self.on_hand[sku] = on_hand - qty


# --- the method that carries the design: plan fully, then commit ---

def fulfill(self, order: Order) -> AllocationPlan:
    demand: Dict[Sku, int] = {}
    for line in order.lines():
        if line.qty <= 0:
            raise InvalidQuantity(line.qty)
        self.catalogue.require(line.sku)
        demand[line.sku] = demand.get(line.sku, 0) + line.qty   # duplicate lines coalesce FIRST

    plan = AllocationPlan()
    for sku, wanted in demand.items():
        remaining = wanted
        for wh in self.strategy.order(list(self.warehouses.values()), sku, wanted):
            if remaining == 0:
                break
            take = min(wh.availableOf(sku), remaining)
            if take == 0:
                continue
            plan.add(wh.id, sku, take)
            remaining -= take
        if remaining > 0:
            raise InsufficientStock(sku, wanted - remaining, wanted)   # NOTHING mutated yet

    for warehouse_id, sku, qty in plan.entries():                       # commit phase
        self.warehouses[warehouse_id].issue(sku, qty)
    return plan
```

**Edge cases covered:** empty order (empty plan) · two lines for the same SKU (coalesced *before* planning — planning them independently double-books the same units) · zero/negative line quantity · unknown SKU vs. unstockable SKU vs. full warehouse, all distinct · a SKU stocked at zero warehouses · a line satisfied exactly by the last warehouse consulted · the plan is complete before any mutation, so the throw path leaves state untouched.

**Trace — order `2×A, 5×B, 1×C`; BLR-1 has `A:3, B:2`, BLR-2 has `A:0, B:4`, neither carries C**

| Step | State |
|---|---|
| demand | `{A:2, B:5, C:1}` |
| A | BLR-1 gives 2, remaining 0 → `plan{BLR-1: A:2}` |
| B | BLR-1 gives 2, BLR-2 gives 3, remaining 0 → `plan{BLR-1: A:2 B:2, BLR-2: B:3}` |
| C | BLR-1 has 0, BLR-2 has 0, remaining 1 → **raise InsufficientStock** |
| after | `onHand` unchanged everywhere — A and B were never deducted |

That last row is the whole point. Deducting line-by-line would have shipped 2 A's and 5 B's for an order that must not ship at all.

## 5. Design decisions

| Decision | Alternative rejected | What it gives up / costs |
|---|---|---|
| `Sku` value object + `Catalogue` as data | `enum ProductSKU` | The enum makes the catalogue a **compile-time artifact**: onboarding a SKU means a code change, a build and a deploy, and a merchandising tool cannot do it at all. The typo argument is real, but `Catalogue.require()` solves it *and* yields a proper `UnknownSku` instead of a `KeyError`. |
| `AllocationPlan` as its own object | Mutating `order.fulfillingWarehouse` | Costs one class. Buys atomicity — a plan is a thing you can build, validate and **discard**. A field on the request object cannot be discarded, and conflates "what was asked for" with "what happened". |
| Two-phase plan-then-commit | Deduct as you go, roll back on failure | Slightly more memory. Buys correctness with no compensating writes — rollback is where atomicity bugs live. |
| **One** pattern: Strategy for allocation ordering | Factory for warehouses, Singleton for the service | Warehouse construction is a constructor call; a Factory buys nothing. A Singleton makes the service untestable and hides a global. Strategy earns its keep because "which warehouse first" is a stated business axis that changes independently of the fulfilment algorithm. |
| `Warehouse` owns capacity **and** the capacity rule | Service reads `warehouse.capacityMap` and compares | Tell-Don't-Ask: the object holding the numbers makes the decision. Otherwise every new caller re-implements the ceiling check, and they will drift. |
| `SkuNotStockedHere` separate from `CapacityExceeded` | `capacity.getOrDefault(sku, 0)` | One extra exception type. Buys a caller who can distinguish "this warehouse doesn't carry that" from "this warehouse is full" — different operational responses. |
| No pattern at all for `Warehouse` / `Catalogue` | Repository, Factory, Builder | Two maps and a set. Adding indirection here is cost with no variation point behind it. **Why no pattern is the right answer here.** |

## 6. Extensibility — each lands on a seam

- **"Allocate from the warehouse nearest the customer."** New `AllocationStrategy` implementation. `fulfill` is untouched.
- **"Reserve stock for 15 minutes before fulfilling."** `Warehouse` gains `reserved: Dict[Sku,int]`; `availableOf` becomes `onHand - reserved`. Every caller already goes through `availableOf`, so the planner needs no change — and `AllocationPlan` becomes the reservation token.
- **"Support stock transfers between warehouses."** `InventoryService.transfer(from, to, sku, qty)` = `issue` then `receive`; both rules already live in the `Warehouse` that owns them.
- **"Warehouses have a total-volume limit, not just per-SKU."** `Warehouse.canAccept` gains a second check. No caller changes — precisely because nobody outside `Warehouse` ever reads its capacity.

## 7. Concurrency

**Category: correctness — check-then-act.** `availableOf()` in the planning phase and `issue()` in the commit phase are separate reads and writes. Two concurrent orders can both plan against the same last unit and both commit, driving `onHand` negative or throwing mid-commit *after* half the order has already been deducted — which breaks the atomicity guarantee the plan/commit split was built to provide. `receive` has the symmetric race against the capacity ceiling.

**Smallest primitive that works:** a single `threading.RLock` on `InventoryService`, held across the **whole** of `fulfill` — plan and commit together. Per-`Warehouse` locks are the tempting refinement and are wrong as a first move: an order spanning warehouses needs all of them, and acquiring them in arbitrary order deadlocks. If per-warehouse locking is later needed for throughput, acquire in a **total order by warehouse id**.

**Where the synchronization lives:** in `InventoryService.fulfill` and `InventoryService.receive` — the orchestrator, because atomicity is a cross-warehouse property and only the orchestrator spans warehouses. `Warehouse` stays lock-free and thread-confined behind the service; that is a documented invariant, not an accident.

**Cost:** the service becomes a single serialization point — all fulfilment is effectively serial, and a slow allocation strategy blocks every other order. Acceptable while allocation is in-memory arithmetic; the moment it does I/O this lock is the bottleneck, and the design must move to per-warehouse locks with ordered acquisition, or to optimistic CAS on per-SKU counters with a bounded retry loop.
