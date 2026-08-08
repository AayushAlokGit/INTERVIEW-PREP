# S006 — Configurable Validations Framework (Rippling, Unified Churn)
**Strength: 4.0 — Primary Customer Obsession story**
**Use for:** Customer obsession / above and beyond for a customer, fixing a class of problem instead of an instance, reducing toil, taking on unprioritized backlog work, designing for configurability, simplification (one source of truth)

**Amazon LPs:** Customer Obsession *(primary)* · Ownership · Insist on the Highest Standards · Dive Deep *(secondary)* · Invent and Simplify *(secondary)*

---

## Spoken Delivery (say this version out loud)

At Rippling I worked on Unified Churn — the product that handled a customer offboarding from one of Rippling's products. A customer representative would raise the churn request and follow it through to completion.

The problem was that there were no validations at the point of request. The system accepted every churn request that came in. But eligibility criteria — things like the company having pending dues — were only checked much later, deep inside the processing pipeline. So a request that could never succeed was accepted anyway, and then quietly got stuck in an in-progress state. A normal churn completed in about a day, so the rep would sit and watch it for three days or more before concluding something was actually wrong and filing a support ticket. And the whole time it sat there, the customer was still being billed for a product they had explicitly asked to leave.

Then it came to me. An engineer — usually me or my mentor — had to go debug that specific request by hand, figure out which criterion had failed, and manually write the reason back into the support ticket. Every single time it happened.

The volume was low, under ten churns a month, and that's exactly why it had been sitting in the backlog without anyone prioritizing it. It was easy to keep deferring.

When I looked at the logs on the stuck requests, failing validations were clearly the cause. But the conclusion I drew was that the reporting wasn't the real defect — the real defect was that a request which couldn't possibly succeed had been accepted in the first place. The failure was being surfaced in the wrong *place*, at a point where nobody who could act on it would ever see it.

So instead of adding alerting or better error messaging on the stuck state, I moved validation to the point of entry. I built an API endpoint called when the churn form was submitted, which ran the checks and returned any failures to the rep before the request was ever created.

The part I care most about is how I did it. The criteria already existed inside the processing pipeline, and the fast path would have been to write a second copy of those checks at the endpoint. I didn't want two copies of the same business rules drifting apart, so I refactored the existing logic out into a registry instead — a dict keyed by the product being churned, mapping to the list of validator functions to run for it. Each validator took the product-level churn request and the company-level request, and returned a boolean plus an error string. One source of truth, called from the entry point, with the pipeline validation kept in place as a second line of defence.

My mentor knew the existing system and oriented me on it, and on design review he made a suggestion I hadn't considered: give the reps the ability to override certain validations, or downgrade their severity. So some criteria stayed non-negotiable hard blockers, and others became tunable from the internal admin UI without needing a deploy, with the overrides persisted as validation-rule records in the database.

I also cleaned up the requests that were already stuck. Those got cancelled, and I made sure a mid-churn cancellation didn't leave the customer in a broken state.

The outcome was that the rep found out what was blocking them at the moment of submission, could go fix it, and watch the validation pass. The entire class of failure — a request accepted successfully and then silently dying days later — was gone by construction, because a request that didn't satisfy its validations could no longer be created at all. No more hand-debugging, and no more customers being billed for days for a product they were trying to leave. No stuck-request tickets came in after it shipped, and the framework was still in active use when I left.

If I'm honest about what I'd do differently: I inferred the customer cost entirely from the tickets I personally worked. I never went and talked to the support or CS people who actually fielded them. If I had, I'd have understood the billing consequence sooner, and I could have made a much stronger case for pulling this out of the backlog earlier than we did.

---

## The Story (STAR)

**Situation:**
Junior engineer at Rippling on Unified Churn, the product handling a customer offboarding from a Rippling product. Customer representatives raised churn requests and followed them through. No validation existed at request time — every request was accepted. Eligibility criteria (pending dues, etc.) were checked only deep in the processing pipeline, so an ineligible request was accepted and then silently stuck in an in-progress state. Normal completion was ~1 day; reps waited 3+ days before filing a support ticket, and the customer continued being billed for the product they'd asked to leave the entire time. Each occurrence required an engineer — usually me or my mentor — to manually debug the request, identify the failing criterion, and hand-write the reason into the ticket. Volume was under 10 churns/month, which is why it sat unprioritized in the backlog.

**Task:**
A senior engineer assigned me to fix it. As one of the two people who debugged those tickets, I already knew the pain first-hand.

**Action:**
Logs on the stuck requests pointed to failing validations. The insight I drew: the defect wasn't poor error reporting, it was that a request which could never succeed was being *accepted*. The failure was in the wrong place — invisible to the only person who could act on it. So rather than adding alerting or better messaging on the stuck state, I moved validation to the point of entry: an API endpoint invoked on churn form submission that ran the checks and returned failures before a request was created. Instead of duplicating the criteria at the endpoint, I refactored the existing pipeline logic into a registry — a dict keyed by product being churned → list of validator functions; each validator received the product-level and company-level churn request and returned a boolean plus an error string. One source of truth, so entry-point and pipeline checks couldn't drift; pipeline validation retained as a second line of defence. My mentor oriented me on the existing system and at design review suggested adding overrides and configurable severity: hard blockers stayed non-negotiable, others became tunable from the internal admin UI without a deploy, with overrides persisted as validation-rule DB records. I also cancelled the already-stuck requests, ensuring a mid-churn cancellation didn't leave customers in a broken state.

**Result:**
Reps got the blocking reason at submission, fixed it, and watched it pass. The class of "accepted request that silently dies days later" was eliminated by construction — such a request could no longer be created. No more manual debugging, and no more multi-day windows of a customer being billed for a product they were leaving. No stuck-request tickets afterwards. The framework remained in active use after I moved on.

---

## Earned Insight
> "A silent failure buried deep in a pipeline isn't a reporting problem, it's a **placement** problem. The check was in the wrong place — somewhere the only person who could act on it would never see it. Move the check to where that person is. And when you move a check, refactor the existing logic into one shared place rather than leaving a second copy behind to drift."

Say this out loud. The move that matters isn't building a validations framework — it's rejecting the obvious fix (report the stuck state better) in favour of the correct one (don't accept the request at all).

---

## Technical Depth Questions (Interviewer Voice)

- **"Why not just add monitoring or alerting on stuck requests? That's much cheaper."**
  **Ans:** It would have cut detection time from three days to minutes, and that's a real improvement — but it treats the symptom. The request was still going to fail; I'd just be finding out faster that I needed to go hand-debug it and explain it to the rep. The actual defect was that the system said "accepted" to a request it already had enough information to know would fail. Once you see it that way, the fix isn't faster detection, it's not accepting it. Alerting would also have left the manual debug-and-explain loop fully intact, which was the recurring engineering cost.

- **"Walk me through what a validator looked like."**
  **Ans:** A function taking two things: the churn request for the specific product, and the company-level churn request — because a company churn could consist of individual churn requests across multiple Rippling products, so some criteria needed the company-wide context, not just the single product's. It ran its check and returned a boolean plus a validation error string, so the caller had both the pass/fail and a human-readable reason to surface.

- **"How did the registry work — how did the endpoint know which validators to run?"**
  **Ans:** A dict keyed by the product type being churned, with each value being the list of validator functions to run for that product. So resolving which checks applied was a lookup on the product being churned, then iterating its validators. That's also what made it extensible — adding a validation for a product meant writing a function and registering it, not touching the endpoint logic.

- **"The criteria already existed in the pipeline. Did you duplicate them or move them?"**
  **Ans:** Refactored, deliberately — that was my call. The fast path was to write a second copy of the checks at the entry point, but then you have the same business rules in two places, and the moment someone changes one and not the other, you're back to accepting requests that will fail downstream. So I pulled the existing pipeline logic out into the registry and had both callers use it.

- **"Did the pipeline keep validating after you added the entry-point check?"**
  **Ans:** Yes. Deliberately belt-and-braces. The entry point is the right place for the user-facing feedback, but I didn't want correctness to depend on every future write path remembering to go through that endpoint. Since both were calling the same registry after the refactor, keeping the pipeline check cost essentially nothing — it wasn't a second implementation to maintain.

- **"You made it impossible to submit a request that failed validation. What if a validation was itself too strict or buggy — haven't you now blocked legitimate churns?"**
  **Ans:** That's exactly the risk, and it's what the configurable severity was for. Some criteria were genuinely non-negotiable hard blockers, but others could be overridden or downgraded in severity from the internal admin UI, with the override recorded as a validation-rule record in the database. So if a rule turned out to be too strict, the escape hatch was a config change by internal staff rather than an emergency deploy. That capability came out of my mentor's design review — he raised the override case and I built it in.

- **"How did the rep experience the errors — all at once, or one at a time?"**
  **Ans:** The churn flow was multiple pages, and validation errors surfaced before submitting each form in the flow. So the rep saw the failures relevant to the step they were on, at that step — contextual to what they'd just entered rather than one aggregate wall of errors at the end.

- **"What did you do about the requests that were already stuck when you shipped?"**
  **Ans:** Cancelled them. The important part was that these were mid-churn, so I had to make sure cancelling didn't leave the customer in a broken or half-offboarded state. It wasn't enough to stop new bad requests — the ones already sitting there were the customers actually being harmed.

- **"Why was there no validation at request time to begin with?"**
  **Ans:** Honestly, I don't have a documented reason — this was hypergrowth-era Rippling and my read is that the churn flow was built to work for the happy path, with the eligibility checks living where the processing logic needed them rather than where the user was. Nobody designed the silent-failure behaviour; it fell out of the checks only existing downstream.

---

## Behavioral Follow-Up Questions (Interviewer Voice)

- **"How did you know this mattered to the customer? Did you talk to them?"**
  **Ans:** Not directly, and this is the honest weak spot. I inferred the cost from the tickets I personally debugged — I was one of the two engineers on the receiving end, so I saw every instance, and I could see the request had been sitting for days and that the customer was still being billed for a product they'd asked to leave. What I should have done is go talk to the support and CS people who fielded those tickets. If I had, I'd have understood the billing consequence earlier and made a stronger case for prioritizing it.

- **"This was under 10 churns a month. Was it worth fixing?"**
  **Ans:** Low frequency, high cost per occurrence. Each one meant a customer stuck for days, still being billed, plus an engineer doing manual forensics and hand-writing an explanation. And the low volume is precisely why it stayed in the backlog — it never looked urgent enough on any given week to displace something else. I'd rather be the person who fixes the unglamorous thing nobody would have blamed me for ignoring. It also wasn't going to get cheaper: every new validation added downstream was another way to silently strand a request.

- **"It was in the backlog and unprioritized — did you push for it, or were you told to do it?"**
  **Ans:** A senior engineer assigned it to me. I want to be accurate about that rather than claim I championed it out of the backlog. What I did bring was first-hand knowledge of the cost from working the tickets, and the decision about *how* to fix it — the choice to move validation upstream rather than improve reporting on the stuck state was mine.

- **"What did your mentor contribute versus what was yours?"**
  **Ans:** He knew the existing system and oriented me on it, and he reviewed my design. His concrete contribution was the override and severity-configuration idea — he raised that reps would need a way to override certain validations or reduce their severity, which I hadn't accounted for, and it turned out to be the thing that made a hard gate safe. The diagnosis, the decision to move validation to the point of entry, and the call to refactor rather than duplicate the existing checks were mine.

- **"What would you do differently?"**
  **Ans:** Two things. First, talk to the support and CS teams instead of inferring the customer cost from my own ticket queue — I'd have understood the billing impact sooner and argued for prioritization earlier. Second, I shipped it and never instrumented the outcome. I can tell you the failure class was eliminated by construction, because such a request can't be created anymore. But I never measured whether the severity-override capability was actually being used — so I can't tell you whether that part of the design earned its complexity or was just a knob nobody turned.

- **"Did anyone extend it after you?"**
  **Ans:** I don't know — I'd left Rippling by then. It was designed so that adding a validation meant writing a function and registering it against a product rather than modifying the endpoint, and it was in active use when I moved on, but I can't claim extensibility was proven by someone else actually doing it.

---

## Watch Out For

- **Lead with the customer, not your toil.** Open on the rep watching a request sit stuck for days while the customer keeps getting billed for a product they asked to leave. The version that opens with "I was tired of debugging these tickets" is a Frugality/Ownership answer, not a Customer Obsession one. Same facts, weaker LP.
- **The hero beat is rejecting the obvious fix.** Better error reporting or alerting on the stuck state was the cheap, natural fix. You went upstream instead, because a request that can't succeed shouldn't be accepted. If the story collapses into "I built a validations framework," you've lost the judgment signal.
- **Never quote a percentage.** The old resume line claimed "20% fewer churn-related tickets." That was an unmeasured estimate and it has been removed from the resume. Do not reintroduce it in any form. Impact is *mechanism* — a class of failure eliminated by construction — not a measured delta. With sub-10/month volume, a percentage invites arithmetic that makes you look both small and imprecise.
- **Attribution, in both directions.** Refactor-over-duplicate was **yours** — own it, it's the best technical call in the story. Override/configurable severity was **your mentor's** design-review suggestion — credit it unprompted. Volunteering that reads as Earn Trust; getting caught reversing it does real damage.
- **Multi-page validation is not a flaw.** Errors surfaced per page in a multi-step flow — that's contextual validation and it's correct. Don't apologize for it or offer it as your what-I'd-do-differently.
- **Oldest, least precise story.** If drilled past real recall: *"This was about four years ago and it's the work I remember least precisely — the shape was X, but I'd rather tell you what I'm confident about than reconstruct the details."* That's a survivable answer. Inventing architecture is not.
- **Known unknowns — say "I don't know" cleanly.** Whether anyone added validators after you left; whether the severity override was ever exercised; why validation was absent originally. All three cost nothing to concede and cost the question to guess at.
