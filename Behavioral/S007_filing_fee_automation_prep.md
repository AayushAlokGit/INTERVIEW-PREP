# S007 — Payroll Filing-Fee Automation (Rippling)
**Strength: 4.5 — Your only fully-defensible number**
**Use for:** Biggest business impact, using data to make a case / change a priority, digging in to size a problem, acting before a window closed, working on code where mistakes are expensive, automating a manual process that kept failing

**Dimensions:** business impact *(primary)* · going and getting the data *(primary)* · acting on a closing window · cost/revenue · delivery
*(Amazon LP mapping → `amazon_lp_prep.md`)*

> ⚠️ **NOT a Think Big story. NOT "a problem nobody asked me to look at."** Your mentor surfaced the gap in conversation; you sized and fixed it. Say so plainly — the sizing is the impressive part and it is entirely yours.

---

## Spoken Delivery (say this version out loud)

At Rippling, when a customer offboarded from our payroll product, if they met certain additional criteria they owed a final flat $1,000 filing fee — that covered filing work Rippling still had to do on their way out. Collecting it was a manual step: someone on the ops team was supposed to notice an eligible churn and reach out to the customer. When churn volume spiked, that step simply got forgotten.

My mentor raised this with me in a conversation — he suspected we were losing money on it. I want to be clear that he surfaced the problem, not me. What nobody had done was put a number on it.

So I went and queried the database directly rather than estimating. I looked for customers with existing churn requests who met the fee eligibility criteria and had never been charged the fee. It came back as 45 customers, at $1,000 each — $45,000 sitting uncollected.

I brought that number back to my mentor, and that number is what turned an offhand remark into actual work. It also surfaced something I hadn't expected: the leak was time-sensitive. Once a customer had *fully* churned out of payroll, there was no billing relationship left and that money was gone permanently. Only the customers still mid-churn were recoverable. So every week we didn't act, recoverable revenue was quietly converting into unrecoverable revenue.

That split the work in two. First, stop the leak going forward. The churn process was already an automated sequence of steps, so I added a step to it that generated the filing-fee invoice automatically for any eligible payroll churn — which took the human out of the loop entirely, so a high-volume week couldn't cause missed revenue anymore.

Second, recover what was still recoverable: backdated invoices for those 45 in-progress customers. This was real customer money, so I didn't make that call myself — I took it to the PMs and got sign-off on retroactively invoicing them.

On safety, because this is billing code and mistakes here are expensive: invoice creation was idempotent, and the actual collection was handled by separate billing infrastructure that had its own idempotency handling. So a re-run couldn't double-charge anyone. And deliberately, my code only *generated* an invoice — the customer still had to pay it themselves. Nothing auto-charged a card. I also built the admin-facing breakdown of the filing-fee line on the invoice, so the $1,000 wasn't an unexplained charge to whoever was looking at it.

The result was $45,000 in backdated invoices generated and charged across those 45 customers, and the manual collection step eliminated going forward. Nothing broke in production, and it was still in use when I left.

The thing I'd take from it is that the highest-leverage work wasn't the code. It was spending an afternoon on queries to turn "I think we're losing money here" into "45 customers, $45,000, and it's shrinking every week." The number is what made it real.

---

## The Story (STAR)

**Situation:**
Junior engineer at Rippling. Customers offboarding from Rippling's payroll product owed a final flat $1,000 filing fee if they met certain additional criteria — covering filing work Rippling still performed on their way out. Collection was manual: ops was supposed to notice an eligible churn and reach out to the customer, and under high churn volume that step got forgotten. My mentor raised in conversation that he suspected this was costing money. Nobody had quantified it.

**Task:**
Determine the actual size of the leak, and if it justified the work, fix it. Not an assigned ticket — it came out of a conversation.

**Action:**
Queried the database directly instead of estimating: customers with existing churn requests, meeting the fee eligibility criteria, never charged. Result: 45 customers × $1,000 = $45,000 uncollected. Brought the number to my mentor — that's what converted the remark into work. The number also exposed urgency: once a customer fully churned out there was no billing relationship left, so that revenue was permanently lost; only mid-churn customers were recoverable, and the recoverable pool shrank every week. Split into two pieces of work. (1) Stop the leak: the churn process was an automated sequence of steps, so I added a step generating the filing-fee invoice automatically for eligible payroll churns, removing the human from the loop. (2) Recover what remained: backdated invoices for the 45 in-progress customers. Because this was real customer money, I got PM sign-off on retroactive invoicing rather than making the call myself. Safety: invoice creation idempotent; collection handled by separate billing infrastructure with its own idempotency, so re-runs couldn't double-charge; and by design the code only generated an invoice the customer paid themselves — no automatic card charges. Also built the admin-facing breakdown of the filing-fee charge so the amount wasn't an unexplained line item.

**Result:**
$45,000 in backdated invoices generated and charged across the 45 recoverable customers. Manual collection eliminated — eligible churns now generate the fee invoice as part of the automated sequence, so volume spikes can't cause missed revenue. No production incidents. Still in use when I left Rippling.

---

## Earned Insight
> "An offhand remark becomes a priority the moment somebody puts a number on it. My mentor already suspected we were leaking money — what was missing wasn't the insight, it was the quantification. An afternoon of queries turned 'I think this is a problem' into '45 customers, $45,000, and the recoverable share is shrinking every week,' and that's what made it obviously worth doing. When a leak is time-sensitive, sizing it *is* urgency work."

Say this out loud. The hero move is the query, not the automation.

---

## Technical Depth Questions (Interviewer Voice)

- **"How did you arrive at $45,000?"**
  **Ans:** Directly from queries I ran against the database. I looked for customers who had an existing churn request, met the eligibility criteria for the payroll filing fee, and had never been charged it. That returned 45 customers. The fee was a flat $1,000 each, so $45,000. It's not an estimate or a projection — it's a count times a fixed fee, and the invoices were subsequently generated and charged.

- **"Is that $45,000 annual or one-time?"**
  **Ans:** One-time. It's the accumulated uncollected balance at that moment, recovered via backdated invoices. It's not a run-rate figure. Separately, the automation prevents future leakage, but I never measured that ongoing amount, so I don't quote a number for it.

- **"Did you actually recover all of it? What about customers who'd already left?"**
  **Ans:** Not all of the total leak, no — and this is the honest limitation. Customers who had *fully* churned out of payroll no longer had a billing relationship with us, so there was nothing to invoice against and that money was permanently gone. The $45,000 was specifically the cohort still mid-churn, which is what made it recoverable. That's also what created the urgency: the longer we waited, the more of that pool crossed over into unrecoverable.

- **"How did the automation hook into the churn process?"**
  **Ans:** The payroll churn was already an automated sequence of steps executed in order. I added a step to that sequence which created the filing-fee invoice when the customer met the eligibility criteria. So it became part of the churn's own execution path rather than a separate job or a reminder for someone to act on — that's what took the human out of the loop.

- **"How did you determine the fee amount for a given customer?"**
  **Ans:** It was a flat $1,000 — no per-jurisdiction or per-filing calculation. The logic was in the eligibility, not the amount: the customer had to be on payroll and satisfy a few additional criteria to be liable at all. If they qualified, the amount was fixed.

- **"This is billing code. How did you make sure you didn't double-charge anyone?"**
  **Ans:** Two layers. Invoice creation on my side was idempotent, so running it again for the same customer didn't produce a second invoice. And the actual charging was handled by separate billing infrastructure that had its own idempotency handling — I wasn't reimplementing collection. Beyond that, the design limited the blast radius on purpose: my code generated an invoice, and the customer paid it themselves. There was no automatic charge to a payment method, so the worst case was an incorrect invoice a human would see, not money silently moved.

- **"You retroactively invoiced 45 customers. Who decided that was okay?"**
  **Ans:** Not me — that's a customer-facing money decision, not an engineering one. I took it to the PMs and got sign-off on backdating invoices for the in-progress cohort. My input was the data: who was eligible, how much, and the fact that the window to recover any of it was closing. The decision about whether to actually bill them was theirs.

- **"What was the admin-facing UI?"**
  **Ans:** To be accurate about scope, it was the breakdown of the filing-fee charge on the invoice rather than a full admin console — so whoever was looking at the invoice could see what the $1,000 line item was and why it was there, instead of an unexplained charge. Given we were backdating invoices to churning customers, an unexplained $1,000 would have generated exactly the support conversation we were trying to avoid.

- **"Why was the manual step failing in the first place?"**
  **Ans:** It wasn't a broken process, it was an unowned one under load. Ops was supposed to notice an eligible churn and reach out to the customer, and when the volume of incoming churn requests was high that step got dropped. Which is the argument for automating it rather than reminding people harder — the failure rate scaled with exactly the volume you'd least want it to fail at.

- **"Did anything break?"**
  **Ans:** No production incidents. I'd attribute that mostly to the two design choices rather than luck: idempotent invoice creation, and never auto-charging — the customer always paid the invoice themselves, so there was a human between my code and anyone's money.

---

## Behavioral Follow-Up Questions (Interviewer Voice)

- **"Whose idea was this?"**
  **Ans:** My mentor's — he raised in conversation that he suspected we were losing money on uncollected filing fees. I don't want to claim I discovered it. What I did was decide that "we're probably losing money" wasn't actionable and go find out how much. The sizing, the recovery approach, and the automation were mine.

- **"What made you go query the database rather than just estimate it?"**
  **Ans:** Because an estimate wouldn't have changed anything. It had been a known suspicion for a while and it hadn't been prioritized, and a rough guess would have been just as easy to defer. A real count with a real dollar figure is a different kind of argument — and as it turned out, running the query is also what surfaced the thing nobody had realised, which is that the recoverable share was shrinking over time.

- **"Was this in your assigned scope?"**
  **Ans:** It came out of a conversation with my mentor rather than off a backlog. Once I brought back the number it became real work with PM involvement, so it wasn't skunkworks — but it started as me deciding to spend an afternoon quantifying something instead of leaving it as a suspicion.

- **"Was there pushback?"**
  **Ans:** Not much, honestly — and I think the number is why. Once it was "45 customers, $45,000, and part of it becomes unrecoverable every week we wait," there wasn't really a counter-argument to make. That's the general lesson I took: most of the persuasion work happens before the conversation, in getting the data.

- **"How did you think about the customer side of retroactively invoicing someone who's leaving?"**
  **Ans:** It was a legitimately owed fee for filing work Rippling actually performed, not a penalty — so the fairness question was about *how* it was communicated, not whether it was owed. That's part of why I built the invoice breakdown: a $1,000 line item on a departing customer's invoice with no explanation is the version that generates a dispute. And I deliberately didn't want the decision to sit with me, which is why the PMs signed off.

- **"What would you do differently?"**
  **Ans:** Two things. I never went back and measured the ongoing prevented leakage after the automation shipped — I can prove the $45,000 recovery, but I can't tell you what the automation saved per quarter afterwards, and that's the more valuable number. And knowing that the recoverable pool was shrinking, the query was worth running the moment the suspicion existed. The delay between "someone suspects this" and "someone counts it" was itself costing money.

---

## Watch Out For

- **Lead with the number and the fact that you went and got it.** Not "I automated a manual billing process." The interviewer's interest is that you converted a vague suspicion into 45 × $1,000 and a closing window.
- **Derive the number, don't recite it.** 45 eligible customers × $1,000 flat fee = $45,000, from queries you ran. Walk the arithmetic if asked. This is the one number in your whole story set that survives arbitrary drilling — contrast it deliberately with the $156K in S001, which still needs auditing before you quote it anywhere.
- **Concede the discovery, immediately and unprompted.** "My mentor raised it; I quantified it." Costs you nothing, and volunteering it reads as Earn Trust. Do **not** deploy this story for "a problem nobody asked me to look at" or "work outside your assigned scope" — those stems will pull the discovery question straight out of you.
- **One-time, not annual.** And volunteer that the fully-churned customers were unrecoverable. "So did you get all of it back?" should never be a question that catches you.
- **You never auto-charged anyone.** Code generated invoices; customers paid them. Have this ready, because "you retroactively charged 45 departing customers" is a natural place for an interviewer to press on judgment.
- **No failure beat exists — do not invent one.** Nothing broke. If pushed on what went wrong, go to the honest limitations: the unrecoverable cohort, and never measuring ongoing prevented leakage. Both are real.
- **Keep the UI claim honest.** The invoice breakdown for the filing-fee charge, not an admin console.
- **Don't overstate seniority.** You were junior, your mentor surfaced the problem, PMs owned the customer decision. The story is strong precisely because the *quantification* was yours — inflating the rest weakens it.
