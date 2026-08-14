# Shopping Cart with Promotions — Design Sheet

## 1. Requirements

1. One shopping cart per user. No capacity limit on shopping cart. No size constraints on cart.
2. User can add items to a cart.
3. User can remove items from the cart. If item does not exist then throw error.
4. Cart has promotion support. A promotion is a rule that reduces the total price of cart. Three kinds of promotions are in play: percent-off a specific item (e.g. 20% off any Book), buy-N-get-M-free on the same item (buy 2 get 1 free), and cart-level threshold (spend over $100, get $10 off). Promotions are configured by the business and supplied to you. The rules are applied to any cart.
5. Read the final price of the cart after applying promotional rules.

## Out of Scope

1. UI
2. Durability of shopping cart

**Language for implementation:** pseudo-code

## 2. Entities & Relationships

- `PromotionRule`
- `Item` (id, name, unitPrice)
- `ShoppingCart` (Map<Item, quantity>, List<PromotionRule>)

**Ownership / entry point:** the caller talks to `ShoppingCart`. `PromotionRule` applies to a `ShoppingCart` — it accepts the cart's current map of items and returns the deduction that rule generates.

## 3. Class Design

```
class PromotionRule:
  + getTotalPriceDeduction(Map<Item, int> cartItems) -> double

class Item:
  - name
  - price
  - id
  + getName() -> string
  + getPrice() -> double

class ShoppingCart:
  - Map<Item, int>
  - List<PromotionRule>
  - getTotalPromotionalRulesDiscount() -> double
  + addItem(Item)
  + removeItem(Item)      // throws exception if item not present
  + getCartTotalPrice() -> double
```

**Clarifications given when probed:**
- `addItem` increments the quantity of the item in the cart map. `removeItem` checks if the item is present in the map, then reduces quantity, else throws an exception.
- `Item.getName()` is used inside `getTotalPriceDeduction` of the promotion rule classes, to check the item type.
- `PromotionRule` is an interface; there will be implementations of it which access the shopping cart items and return the discount according to that particular rule.

> Round stopped here by request — phase 4 (implementation) was never written. Everything above is his, verbatim.

---

# Ideal Front Half (writable in the same 22 minutes)

Judged against the `/lld-sprint` rubric. Front-half readiness: **3/5**.

## 1. Requirements

**FR1** — Caller is a single user session talking to `ShoppingCart`; `ShoppingCart` is the orchestrator.
**FR2** — Operations: `addItem(item, qty)`, `removeItem(item, qty)`, `total()`.
**FR3** — `qty` must be > 0. Removing an item not in the cart, or removing more than held, is illegal.
**FR4** — Cart is mutable indefinitely; no expiry. Checkout is out of scope, so the cart has no terminal state.
**FR5** — *Failure convention:* illegal mutations throw `IllegalArgumentException`. Read operations never throw. Held for every signature.
**FR6** — Three promotion kinds: percent-off-item, buy-N-get-M-free-same-item, cart-threshold. **All rules evaluate against the original subtotal and stack additively; total is floored at zero.** Money is integer minor units, not `double`; percent-off rounds half-up to the cent. Single currency.
**FR7** — Single-threaded. Confirmed, not assumed.
**FR8** — **Out of Scope:** UI, persistence, auth, payment, inventory/stock, tax, shipping, promo codes and eligibility, multi-currency, checkout.

> *What this buys:* FR6 is the whole gap. His design silently picked one stacking policy out of three plausible ones — `getTotalPromotionalRulesDiscount()` sums independent per-rule deductions, which commits to "every rule evaluates against the original subtotal" without ever asking. Stating it takes ten seconds and it is the answer the interviewer is waiting for. FR5 turns one local decision (throw on remove-missing) into a convention that types every signature.

## 2. Entities & Relationships

- `Item` — immutable value object (id, name, unitPrice). Owned by nobody; referenced by cart and rules.
- `ShoppingCart` — **the orchestrator.** Owns `Map<Item,int>` and `List<PromotionRule>`.
- `PromotionRule` — interface. The seam.
- `PercentOffItemRule`, `BuyNGetMFreeRule`, `CartThresholdRule` — three implementations.
- `CartView` — read-only projection the cart passes to rules.

**The entities that exist only to hold a rule:** the three `PromotionRule` implementations. FR6 forces them — each exists solely to own one promotion's parameters and arithmetic. No line-item class: a `Map<Item,int>` entry holds no rule of its own, so wrapping it earns nothing.

> *What this buys:* his list named the seam (`PromotionRule`) but never the things behind it, so at the buzzer no object anywhere held the 20%, the N/M, or the $100 threshold.

## 3. Class Design

```
class Item:                                   // immutable
  - id: string                                (FR6 — rules target items by id)
  - name: string                              (display only)
  - unitPrice: Money                          (FR2)
  + id(): string
  + unitPrice(): Money

interface CartView:                           // read-only, hides the cart's map
  + quantityOf(itemId: string): int
  + subtotal(): Money

interface PromotionRule:
  + discountFor(cart: CartView): Money        (FR6)

class PercentOffItemRule implements PromotionRule:
  - targetItemId: string                      (FR6)
  - percentOff: int                           (FR6)
  + discountFor(cart: CartView): Money

class BuyNGetMFreeRule implements PromotionRule:
  - targetItemId: string                      (FR6)
  - buyN: int, getM: int                      (FR6)
  + discountFor(cart: CartView): Money

class CartThresholdRule implements PromotionRule:
  - threshold: Money, discount: Money         (FR6)
  + discountFor(cart: CartView): Money

class ShoppingCart implements CartView:
  - lines: Map<Item, int>                     (FR2, FR3)
  - rules: List<PromotionRule>                (FR6, injected via constructor)
  + addItem(item: Item, qty: int): void       // throws if qty <= 0              (FR5)
  + removeItem(item: Item, qty: int): void    // throws if absent or qty > held  (FR5)
  + subtotal(): Money
  + totalDiscount(): Money
  + total(): Money                            // max(subtotal - totalDiscount, 0)  (FR6)
```

**Why each rule sits where it does:** quantity legality lives in `ShoppingCart` because it owns the map. Each promotion's arithmetic lives in its own class because that class owns the percentage / N / M / threshold — nothing else can compute it. `ShoppingCart` never branches on rule type; that is the seam. `CartView` exists so the cart hands rules a read-only projection instead of its live internal `Map`.

> *What this buys:* every promotion parameter now has a home, so FR6 is enforceable instead of aspirational. Every field carries the requirement number that forces it, which kills `Item.id`-with-no-caller. And `CartView` closes the `getName()` leak — rules match on `id`, and the cart's map never leaves the cart.

---

## Scoring summary

| Phase | Pace target | Landed at | ± vs target | Messages | Score |
|---|---|---|---|---|---|
| Requirements | 6:00 | 12:46 | +6:46 | 3 | 3/5 |
| Entities & relationships | 10:00 | 15:40 | +5:40 | 2 | 2/5 |
| Class design | 22:00 | 19:43 (fills at 22:21) | +0:21 | 2 | 2/5 |

**Walk coverage:** 3 Hit · 5 Partial · 0 Miss.
Hit — core operations, concurrency posture, explicit Out of Scope (unprompted).
Partial — actors & entry point, rules & legality, lifecycle & terminal states, failure behaviour, multiplicity & variants.

**Out of Scope list produced:** Unprompted — first time in the file's history.
**Orchestrator named:** Only when asked.
**Entity revision passes:** 0.
**Dangling rules:** none.
**Untyped state or signatures:** `Item`'s fields carry no types; `ShoppingCart`'s fields carry no names; `addItem`/`removeItem` carry no return types.
**Untraceable state / unenforceable requirements:** `Item.id` declared and never used (identity matching goes through `getName()` instead). FR4's three promotion kinds have no state in any class — the design as written cannot compute a single promotion.

**Silent assumptions:** promotion stacking order and which subtotal the threshold tests; `double` used for money with no rounding convention; total never floored at zero.

**First-pass completeness:** every phase took exactly one more message than it should have, and message two was always predictable from message one — stacking follow-up after the promotion question, ownership/orchestrator after the entity list, return types and interface-vs-class after the outline. The content was correct when it arrived; it arrived in two pieces.

**The one habit to change:** before submitting any phase, name the single thing an interviewer would have to ask about it — and answer that in the same message.
