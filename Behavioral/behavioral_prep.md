# Behavioral Prep — Aayush Alok
Last updated: 2026-07-17

> Index + quick-reference. Full STAR, technical-depth Q&A, and follow-ups live in each `S00x_*_prep.md` file.

---

## Story Quick Reference

| ID | Title | Strength | Use When |
|----|-------|----------|----------|
| S001 | Zero-Downtime Config Migration | 4.5 | When in doubt — most versatile. Production incident, failure, admitting a mistake, recovery |
| S004 | Regression Coverage from Ambiguity (LLM Agent E2E) | 4.5 | Ambiguity, ownership, GenAI/LLM depth, a feature you owned end-to-end |
| S002 | Knowing When to Concede (Test Runner Removal) | 4.0 | Disagreement with a peer, changing your mind on new info, conceding gracefully |
| S003 | NSP Security Compliance Deployment | 4 | Technical judgment ("read the system first"), large-scale infra, security, hard deadline |
| S005 | Copilot Usage Analytics Dashboard | 4 | Learning unfamiliar tech fast, knowing when to ask for help, self-awareness / a habit you fixed, cross-team dependencies |
| S006 | Configurable Validations Framework (Rippling) | 4 | **Customer obsession / customer impact**, fixing a class instead of an instance, reducing toil, unprioritized backlog work you took on |
| S007 | Payroll Filing-Fee Automation (Rippling) | 4.5 | **Business impact — your only fully-defensible number ($45K = 45 × $1,000)**, using data to make a case, acting before a window closed, expensive-mistake code |

**Two lead stories (4.5):** S001 and S004. When you get to pick, open with one of these.

---

## Coverage Map

| Category | Story |
|---|---|
| Most versatile / "when in doubt" | S001 |
| Production incident + recovery | S001 |
| Failure / learning | S001 |
| Admitting a mistake | S001 |
| Communication under pressure | S001 |
| System reliability / zero-downtime design | S001 |
| Prioritization / sequencing under pressure | S001 (blast-radius sequencing), S003 (parallelizing under deadline) |
| Receiving feedback | S001 (manager's corrective guidance), S002 (peer's feedback → changed course) |
| Disagreement with a senior / peer | S002 (primary), S003 (challenged security decision) |
| Changing your mind on new information | S002 |
| Asking "why" / curiosity before defending | S002 |
| Conceding gracefully / ego vs. right answer | S002 |
| Ambiguity / undefined problem | S004 (decompose before building), S005 (requirements-gathering), S003 |
| Ownership / self-directed / sole owner end-to-end | S004, S005, S003 |
| GenAI / LLM experience | S004 (hands-on: eval vs. E2E split), S005 (analytics *about* Copilot usage) |
| Decomposition / imposing structure on ambiguity | S004 |
| Testing & quality | S004 |
| Technical judgment / "read the system before you design" | S003, S001 (write-path audit) |
| Security / compliance trade-off | S003 |
| Large-scale infrastructure change | S003 |
| Delivering under a hard deadline | S003, S005 |
| Cross-team coordination / dependency management | S003, S005 |
| Stakeholder / leadership communication | S005, S004 |
| Learning unfamiliar tech fast | S005 |
| Knowing when to ask for help | S005 |
| Self-awareness / a habit I changed | S005 (timezone coordination) |
| Business / cost impact | **S007 ($45K recovered — derive it live: 45 × $1,000)**, S001 ($156K/yr — *attribute to your manager, say "forecast," then pivot to mechanism*), S004 (eliminated 1–2 person-days/release) |
| Using data to change a priority / make a case | S007 (primary), S005 (200GB load test → drove Power BI fix) |
| Acting because a window was closing | S007 (fully-churned customers were unrecoverable) |
| **Customer obsession / worked backwards from the customer** | **S006** (primary), S007 filing-fee (business-side), S004 (manual QA as internal customer) |
| Above and beyond for a customer | S006 |
| Fixing a class of problem, not an instance | S006 (primary), S001 (ConfigStore abstraction) |
| Reducing toil / support burden | S006 |
| Taking on work nobody prioritized | S006 (sat in backlog because volume was low) |
| Couldn't meet a commitment | S002 |
| Digging into details to understand a complex problem | S005 (perf investigation), S003, S001 |
| Giving feedback | Script (gap) |
| "Why did you leave Microsoft?" | Script |

---

## Amazon Leadership Principles → Story

> **Why this exists.** Amazon interviewers are each assigned 2–3 LPs and score you against them, not against topics. A loop is 4–5 interviewers × 2–3 LPs = **10–14 probes**, and they debrief together — so a story reused across three interviewers is visible in a way it isn't within one round. 16 LPs, 7 stories: **reuse is expected and normal.** The goal is ≤2 probes per story per loop, not one story per LP.
>
> **Rated by what you can actually defend, not by what fits.**

| # | Leadership Principle | Primary | Backup | Coverage |
|---|---|---|---|---|
| 1 | **Customer Obsession** | **S006** | **S004** (regression = broken UI a customer sees), **S003** (customer data protection + not breaking live traffic), S007 (business-side + billing fairness) | ⚠️ Improving — S006 primary, but use the **Customer-Impact Framing** section below to open S004/S003 customer-forward |
| 2 | **Ownership** | S004 | S005, S003, S007 | ✅ Strong |
| 3 | **Invent and Simplify** | **S006** (registry = one source of truth) | S001 (`ConfigStore` abstraction), S003 (profiles grouped by traffic pattern) | ✅ Good — *label these as simplification, they're already there* |
| 4 | **Are Right, A Lot** | S003 (learning mode instead of guessing) | S002 (updated on evidence) | ✅ Good — *was shaky only because of the numbers; both audited now* |
| 5 | **Learn and Be Curious** | S005 | MCP side project, S002 (asked *why*) | ✅ Strong |
| 6 | **Hire and Develop the Best** | S004 (new grad ramped + added tests off your docs) | Giving-feedback script | ⚠️ Bridge — no formal mentoring, don't pretend |
| 7 | **Insist on the Highest Standards** | S004 | S001 (audit from a checklist, not memory), S006 | ✅ Strong |
| 8 | **Think Big** | S003 (company-wide, 23 subs, two teams adopted your doc) | S007 (turned a suspicion into a funded workstream) | ⚠️ Level-calibrated — see note below |
| 9 | **Bias for Action** | S001 (flag off in 15 min) | S003 (learning-mode deploy), S007 (closing recovery window) | ✅ Strong |
| 10 | **Frugality** | **S007 ($45K — derive it live)** | S001 ($156K — *attribute + mechanism*) | ✅ Good now |
| 11 | **Earn Trust** | S001 (admitted fault *before* having a fix) | S005, S006/S007 (credited mentor unprompted) | ✅ Strongest |
| 12 | **Dive Deep** | S005 (200GB load test → traced to Power BI) | S007 (queried the DB rather than estimating), S003, S001 | ✅ Strongest |
| 13 | **Have Backbone; Disagree and Commit** | **S002 — Backbone variant only** | S003 (challenged a security decision) | ⚠️ Use the re-angled framing, never the default |
| 14 | **Deliver Results** | S003 | S005, S007 | ✅ Strong |
| 15 | Earth's Best Employer | S005 (timezone habit — stopped offloading inconvenience onto another team) | Bridge | Rarely probed for SDE |
| 16 | Success and Scale Bring Broad Responsibility | S003 (security/exfiltration risk), S007 (billing fairness → PM sign-off) | Bridge | Rarely probed for SDE |

### Load-balancing — don't over-serve one story
S001, S003 and S004 each appear against 5+ LPs. In a real loop **cap any story at 2 uses.** If S001 has already carried Earn Trust and Bias for Action, route the next probe to S006/S007 even if S001 fits better. Interviewers compare notes; three interviewers hearing the config migration reads as a one-story candidate.

### The three honest caveats

**Think Big is level-calibrated.** At 3.5 years the bar is "proposed scope past what you were handed," not "redirected strategy." S003 is your best shot — company-wide initiative, 23 subscriptions, and two other teams picked up your documented approach. S007 is adjacent: nobody had sized the leak, and quantifying it created the workstream. Neither is a moonshot; don't inflate them. But *"I don't have one"* scores worse than an honest mid-level answer.

**Hire and Develop is a bridge, not a story.** The real evidence is a new grad ramping and adding tests using documentation you wrote — enabling someone else, which is the substance at your level. Pair with the giving-feedback script. Don't claim mentoring you haven't done.

**Customer Obsession no longer rests on S006 alone — but only if you open correctly.** S006 stays primary (it's the one where the customer is the subject of the story). S004 and S003 are genuine backups *provided* you lead with the customer sentence: for S004, a customer opening the agent and finding the LLM insights don't render; for S003, exfiltration risk on customer data plus the refusal to guess at access rules because a wrong rule breaks live customer traffic. Both framings are true and neither needs new material — see **Customer-Impact Framing** below. The risk that remains isn't coverage, it's **habit**: under pressure you open on the system and never circle back to the person.

---

## Customer-Impact Framing — how to open each story customer-forward

> **Why this exists.** Almost all your work is platform/internal, so your instinct is to open on the *system* — a config store, a security perimeter, a test pipeline. Customer Obsession is your weakest LP and your panel's dimension 3, and the fix is mostly **which sentence you open with**, not new material. Every framing below is true; none requires inventing impact.
>
> **The move:** name the person harmed and what they experienced, *then* the system. One sentence, then proceed as normal.
>
> **Two hard rules.**
> 1. **Never say "luckily there was no customer impact."** It's on your banned list. Containment was your design — low-traffic geo first, blast-radius sequencing — not luck. Say *"contained by design"* and then say how.
> 2. **Don't stretch.** An interviewer can tell the difference between a real customer argument and a retrofitted one. Where the honest answer is "my customer here was internal," say that — it scores better than an inflated claim. The ceilings below are deliberate.

| Story | Customer-forward opening move | Honest ceiling |
|---|---|---|
| **S001** Config Migration | *"Config for CRM analytics was stored per-geo, so a single storage outage took the feature down for every tenant in that geo at once. Shrinking that to island scope meant a future outage could only affect one island's customers instead of an entire region's."* Then: rollout started in the **lowest-traffic geo by design** to cap customer exposure, and when it went wrong you **sequenced recovery by blast radius** — stop new damage, recover existing, fix root cause, verify — because the ordering was about limiting who got hurt. | **Strong, second-order.** The reliability argument is genuine. But you were not talking to customers — don't imply you were. |
| **S003** NSP Rollout | *"The gap was a data-exfiltration risk on resources holding customer data — so this was about customer data protection, not compliance paperwork."* Then the sharper beat: **the reason you refused to write access rules from the code was that a wrong rule breaks live customer traffic.** Learning mode existed specifically so you'd derive rules from real behaviour rather than guess and take features down. "No legitimate traffic disrupted, no features broken" = customers never noticed a company-wide security change, **which was the objective.** | **Strong.** Protecting customer data and not breaking customer traffic are both real and both yours. |
| **S004** LLM Regression Testing | *"A regression meant a customer opening the sales agent and finding the LLM insights simply didn't render — and we shipped weekly, so a bad release reached customers within days."* Then: you **sat with the manual QA vendors to map the flows customers actually use**, and built the **P0 list with PMs and feature owners** — that prioritisation was customer-criticality, not code coverage. | **Strong.** The system's entire purpose is customers not seeing broken features. This is your best non-S006 Customer Obsession framing. |
| **S006** Configurable Validations | Already customer-forward. *"Still being billed for a product they'd asked to leave."* | **Primary story.** |
| **S007** Filing-Fee Automation | Customer here is the **business**, plus a fairness beat: you built the invoice breakdown so a departing customer saw *why* they owed $1,000 rather than an unexplained charge, and you took retroactive invoicing to **PMs** rather than deciding unilaterally what to bill people. | **Moderate.** Revenue-side. Don't pitch it as customer-first; the fairness/communication beat is the honest customer content. |
| **S005** Copilot Dashboard | *"My customer was internal — leadership — and they'd never had visibility into Copilot usage."* The one genuine external thread: you sized the load test to the **largest customer's data volume**, because the worst-case refresh had to hold for the biggest tenant, not the average one. | **Weak — say "internal customer" out loud.** Analytics *about* Copilot usage. Don't inflate. |
| **S002** Test Runner Removal | Second-order and worth stating once: your test runner was **blocking customer-facing services from shipping**, and every mid-deploy failure cost hours of restart. Conceding removal was choosing customer-facing deployment reliability over your own coverage. | **Weak, internal.** State it in one clause and move on; the story's value is Backbone, not customers. |

### Drill this separately
Cover the middle column. For each story, say **only the customer-forward first sentence** — 10 seconds each, seven stories. The failure mode isn't that you lack the argument; it's that under pressure you open on the system and never get back to the person. One pass a day until the customer sentence comes first by reflex.

---

## Retrieval Table — Question Stem → Story → Opener

> **Why this exists.** The Coverage Map above is *category → story*, which needs you to classify the
> question first. In a live round that hop is where you stall. This table is *the sentence you'll
> actually hear* → story → the first thing out of your mouth. One hop.
>
> **How to drill it.** Cover the right-hand columns. Have someone read stems at random, 10 seconds
> each. Say only: story name + opener. Not the story. Forty stems in seven minutes. Repeat until
> it's reflex. Picking the story is a separate skill from telling it — drill it separately.

### S001 — Config Migration
**Opener:** *"During a live config-store migration I caused data loss in production — and the lesson wasn't the bug. It was that my design had no tolerance for an ordinary mistake."*
**Closer:** *"The dangerous sequence is copy → delete → route. The safe one is copy → route → verify → then delete."*

- Tell me about a mistake you made / a time you failed
- Tell me about a production incident you caused or handled
- Tell me about feedback that was hard to hear *(primary — S002 is backup)*
- Tell me about a time you had to admit you were wrong
- Tell me about a time you had to deliver bad news to your manager
- Tell me about a time several things were urgent at once / how do you prioritize *(the blast-radius sequencing: stop new damage → recover existing damage → fix root cause → verify)*
- Tell me about a time you had to act fast under pressure
- Tell me about a reliability or zero-downtime design decision
- What's the biggest thing you've changed about how you work?
- Tell me about a time you took a risk

### S002 — Test Runner Removal — **two versions, pick before you speak**
> ⚠️ This story has **two rehearsed tellings** with opposite emphases. Picking wrong loses the question even though every fact is true. Full decision table in `S002_test_runner_removal_prep.md`.

#### 🔄 Version A — "Changed My Mind" *(the default below)*
> ✅ **Answers:** a time you were wrong · changed your mind · new information changed a decision · wrong about something technical · killed your own work · couldn't meet a commitment
> ❌ **Never for:** disagree and commit · held your ground · pushed back on a senior *(→ Version B)*

**Opener:** *"Someone recommended deleting a service I'd built. I disagreed — and what resolved it wasn't an argument, it was a question I asked him."*
**Closer:** *"I was evaluating a technical event; he was living its operational cost. Understand the operational constraints before you defend the technical decision."*

- Tell me about a disagreement with a teammate / peer / senior engineer
- Tell me about a time you changed your mind
- Tell me about a time you were wrong about something technical
- Tell me about a time you pushed back on someone
- Tell me about a time you had to kill your own work
- Tell me about a time you got new information that changed a decision
- How do you handle conflict on a team?
- **Tell me about a time you couldn't meet a commitment** *(you committed a test runner into the shared pipeline, then retracted the coverage when you couldn't keep it disruption-free — stay on the commitment you dropped, not the peer dynamic)*

#### ⚔️ Version B — Backbone
> ✅ **Answers:** disagree and commit · held your ground · pushed back on a senior person · disagreed but supported it anyway · strongest disagreement at work · *neutral* "a disagreement with a teammate" → **start here**
> ❌ **Never for:** a time you were wrong · changed your mind · killed your own work *(→ Version A)*
> **Amazon LP:** Have Backbone; Disagree and Commit — **your only coverage for it.**
>
> Same story from the other end. Version A leads with *the question I asked* and lands on *changing my mind* — correct for "a time you were wrong," but a **weak Backbone answer**, because the interviewer hears "disagreed, then folded." Front-load the resistance; make the commit the closer.

**Opener:** *"A senior engineer wanted a service I'd built deleted from every geo. I didn't accept that — I pushed back twice, and I made him show me the cost before I'd agree to anything."*
**Closer:** *"Once I was convinced, I didn't just stop objecting — I built the removal plan with him and documented why, so nobody could quietly undo the decision later. Disagreeing is cheap. Committing to the outcome you argued against is the part that counts."*

**The four beats, in order — don't skip 1 or 4:**
1. **Held ground.** He escalated for full removal; I pushed back — the service ran fine in most geos, the failure was transient, a retry had already cleared it. When he escalated a *second* time, I still didn't concede.
2. **Made him justify it.** Asked directly *why* full removal, which surfaced the 3–4hr deploy with a full restart on any mid-deploy failure.
3. **Did the work before yielding.** Evaluated auto-retry (platform-abstracted, not available to us) and passing-geo-only scoping (two main branches for a subset-coverage internal service — overhead unjustified). Conceded on evidence, not on his title.
4. **Committed all the way.** Co-authored the removal plan, documented the rationale so a future engineer wouldn't re-add the landmine, and brought it to my manager. Then on a later rollout with the same engineer, front-loaded the deployment-impact analysis I'd previously skipped.

- Tell me about a time you disagreed with someone and had to commit anyway
- Tell me about a time you held your ground / pushed back on a senior person
- Tell me about a time you disagreed with a decision but supported it
- Tell me about the strongest disagreement you've had at work

> **Never say "I conceded" as the headline.** You conceded *to evidence you demanded and then verified yourself*, and then you owned the outcome. That's the LP.

### S003 — NSP Security Rollout ⚠️ **Under-used — reach for this more**
**Opener:** *"I was handed a company-wide security rollout with a hard two-month deadline, and the highest-leverage thing I did was refuse to design anything until I'd read how the system actually behaved."*
**Closer:** *"Read actual system behavior before you make an architectural decision."*

- **Tell me about a decision you made with incomplete information** ← **PRIMARY. This is the only clean answer in the set.** The beat: you couldn't learn every traffic pattern from the code, so rather than guess at access rules you deployed in *learning mode* and wrote them from observed live traffic. Committing to a rollout while still not knowing.
- Tell me about a time you had to deliver against a hard deadline
- Tell me about the largest-scale change you've made
- Tell me about a time you led something above your level
- Tell me about a time you got blocked and what you did *(the Azure quota block — parallelized into unblocked geos, cost zero calendar time)*
- Tell me about a security or compliance trade-off
- Tell me about a time you rejected the obvious solution
- Tell me about work of yours that had impact beyond your team *(two other teams picked up the doc)*
- Tell me about a time you investigated before building

### S004 — LLM Agent Regression Testing
**Opener:** *"I was asked to 'build something to catch regressions before each weekly release.' That was the entire ask — no spec, no scope. The most important work happened before I wrote a line of code."*
**Closer:** *"When the problem is ambiguous, decompose the space before you build."*

- Tell me about an ambiguous problem / something with no spec
- Tell me about a feature or system you owned end-to-end
- Tell me about your hands-on LLM / GenAI experience *(pair with the MCP/RAG side project)*
- Tell me about a testing or quality decision you made
- Tell me about a time you had to align stakeholders
- Tell me about something you built that others reused
- Tell me about a project you're proud of
- Tell me about a time you had to define the problem yourself

### S005 — Copilot Usage Dashboard
**Opener:** *"I was the sole engineer on a leadership dashboard with a 1.5-month deadline, a 1TB/day telemetry stream, and zero prior experience with any system involved."*
**Closer:** *"Cross-timezone coordination is a logistics problem to solve immediately, not a personal inconvenience to minimize."*

- Tell me about learning an unfamiliar technology fast
- Tell me about a time you asked for help / knew you were stuck
- **Tell me about a habit you changed / a weakness / what you'd do differently** *(the timezone beat — your single best self-aware answer; lead the reflection with it)*
- Tell me about a cross-team dependency
- Tell me about a time you found a problem that wasn't yours to fix *(the ~3hr refresh — you found it, quantified it, drove it; Power BI owned the fix)*
- Tell me about communicating a risk to leadership
- Tell me about a performance problem you investigated
- **Tell me about a time you dug into the details to understand a complex problem** *(the ~200GB load test that reproduced the worst-case initial refresh and traced the ~3hr time to the Power BI layer, not your pipeline)*
- Tell me about work that got shelved or deprioritized

### S007 — Payroll Filing-Fee Automation (Rippling) ⭐ **Your one fully-defensible number**
**Opener:** *"My mentor mentioned in passing that we were probably losing money on payroll offboarding because a $1,000 filing fee was still collected by hand. Nobody had put a number on it, so I went and queried the database — 45 customers, $45,000 uncollected, and a closing window to get any of it back."*
**Closer:** *"An offhand remark becomes a priority the moment someone puts a number on it. The number was the work."*

- **Tell me about your biggest business impact** *(primary)*
- **Tell me about a time you used data to make a case / to change a priority**
- Tell me about a time you dug into something to find out how big it really was
- Tell me about a time you had to move fast because a window was closing
- Tell me about a time you saved the company money / recovered revenue
- Tell me about a time you worked with a PM on a customer-facing money decision
- Tell me about working on something where mistakes were expensive
- Tell me about automating a manual process that kept failing

> ⚠️ **Do NOT use for "a problem nobody asked me to look at" or "work outside my assigned scope."** Your mentor surfaced the gap — you sized and fixed it. Claiming discovery here is a trust risk for zero gain; the sizing is the impressive part and it's entirely yours.

### S006 — Configurable Validations Framework (Rippling) ⭐ **Customer Obsession story**
**Opener:** *"A customer would submit a request to offboard from a Rippling product, get told it went through, and then watch it sit stuck for days with no explanation — because the only thing that could tell them what was wrong was me, manually debugging it."*
**Closer:** *"A silent failure buried in a pipeline isn't a reporting problem, it's a placement problem. Put the check where the person who can act on it will see it."*

- **Tell me about a time you went above and beyond for a customer** *(primary — lead with the rep stuck for days, not with your own toil)*
- **Tell me about how you think about the customer / worked backwards from the customer**
- Tell me about a time you fixed a class of problem instead of an instance
- Tell me about a time you reduced toil or support burden
- Tell me about work you took on that nobody had prioritized
- Tell me about designing for configurability / extensibility
- Tell me about a time you improved someone else's experience of a system
- Tell me about a time you simplified something *(one registry, one source of truth — rather than a second copy of the checks)*

---

### Collisions — who wins when two stories fit

| Question | Primary | Backup | Never |
|---|---|---|---|
| Receiving feedback | S001 | S002 | — |
| Disagreement / conflict | S002 | S003 | — |
| Ambiguity | S004 | S005 (requirements), S003 (design) | — |
| Decision under uncertainty | **S003** | — | S004 — *ambiguity ≠ uncertainty; you resolved that one by asking* |
| Ownership end-to-end | S004 | S005, S003 | — |
| Prioritization | S001 (blast radius) | S003 (parallelizing) | — |
| Hands-on GenAI | S004 | MCP side project | S005 — *analytics about Copilot, not building it* |

### The two genuine gaps — bridge, never dead-end

**Giving feedback to a peer.** No story exists. Say: *"I haven't been in a formal position to give feedback yet — most of mine has been peer code review. My principle is specific and actionable over vague: I'd rather say 'this will fail on null input, here's the fix' than 'this needs improvement.'"* Then give a real code-review example. **Never** say "I've never had to tell someone they were wrong" — it reads as either no reflection or stonewalling.

**Prioritizing between competing projects.** Say: *"At my stage I've mostly executed on a defined backlog — my judgment has been sequencing within a problem, not between projects. That's something I want to develop at the next level."* Then immediately pivot to the S001 blast-radius sequencing so the interviewer still gets evidence. **Never** go hypothetical.

> **Banned phrases — each one loses the question outright:**
> "I do not have any such experience." · "I've never had such a moment." · "Not sure how to answer."
> · "Hypothetically, I would…" · "Luckily, there was no customer impact."
>
> **The bridge, memorized:** *"Let me think for a second about the sharpest example."* Then reach for
> the nearest true thing and say why it's adjacent. Silence is recoverable. "I don't have one" is not.

---

## Stories

### S001 — Zero-Downtime Config Migration *(Strength 4.5 — Lead story)*

**Situation**: D365 Sales Hub microservices ran Spark jobs for CRM analytics off a config store on Azure Table Storage — geo-scoped, high COGS, large blast radius. A geo-level outage took down configs for every island in that geo.
**Task**: Zero-downtime, live migration to PPMS (Cosmos DB, island scope) — $156K/yr savings, shrink the fault boundary.
**Action**: Background service drained data (copy → completion flag → cleanup); write paths checked the flag to route to PPMS; per-geo feature flag; started low-traffic. Within 15 min noticed writes still landing in Table Storage — I'd missed the flag check on one write path, and cleanup had already run, so those writes were going to a store nothing read from. Flipped the flag off to stop the bleeding, escalated to my manager immediately (admitted fault before I had a fix). But flag-off pointed every service back at a Table Storage the migration had already drained — so I restored it by copying the data back out of PPMS (authoritative at that point), verified every value, then centralized routing behind a `ConfigStore` abstraction and audited every write path before resuming.
**Result**: Loss contained to one low-traffic geo; full recovery in 15 min; no meaningful customer impact. Migration completed zero-downtime across all geos. $156K/yr forecast COGS reduction *(manager's figure — attribute it, see below)*. Blast radius geo → island.

**Earned Secret**: Dangerous sequence is copy → delete → route. Safe sequence is copy → route → verify → **then** delete. If you delete before verifying, you've destroyed your own fallback — which is exactly what made flag-off insufficient. Also: centralize routing behind one abstraction *before* migrating, and audit from a checklist, not memory.
**Watch out for**: **The $156K — attribute it unprompted.** *(Audited 2026-08-08.)* It's your manager's COGS figure from an email to senior leadership, attributed to your workstream, and it's a **forecast**, not audited realized savings. You never saw a before/after cost number and didn't do the math. So say: *"my manager did the COGS analysis and reported $156K/yr to leadership — that's his forecast, not my calculation. What I can tell you is why the cost dropped."* Then the mechanism: **you were paying for one dedicated Table Storage resource per geo; PPMS was an existing shared internal platform other teams already ran on, so the migration decommissioned your whole per-geo fleet and moved onto already-funded infrastructure.** The saving is resource elimination, *not* a cheaper database — which also pre-empts "isn't Cosmos more expensive per-GB?" (it is; that's why the mechanism matters). Full Q&A in `S001_config_migration_prep.md`.
**Watch out for**: **Keep recovery physics straight — three beats, in order.** (1) Flag off *stopped* loss and *created* the empty-store problem; (2) copying PPMS → Table Storage *restored* the drained store; (3) `ConfigStore` abstraction fixed the root cause. Don't merge them — claiming to recover the buggy path's writes from PPMS doesn't hold, since PPMS never had them. Open follow-up still to settle: what happened to the buggy path's writes when you copied PPMS over the top? "Why didn't tests catch it?" → test env didn't replicate the low-frequency write path; own the gap. Don't let it sound like your manager bailed you out — he gave direction, you executed.

---

### S004 — Regression Coverage from Ambiguity (LLM Agent E2E) *(Strength 4.5 — Lead story)*

**Situation**: Mid-level engineer on a team shipping an LLM-based sales opportunity agent on a weekly cycle. Asked to "build a regression-catching mechanism before each release" — no spec, no scope, no infra. Real stakes: regressions meant broken UI (e.g. LLM insights not rendering).
**Task**: Turn an undefined ask into a concrete automated system, owning scope + architecture + stakeholder alignment.
**Action**: Resisted writing tests first. Sat with the manual QA vendors to map real critical flows. Made the key call: split nondeterministic from deterministic — LLM output *quality* → dedicated eval system; *deterministic layer* (agent setup, UI rendering, user access) → my E2E tests. Pragmatic edge-case line: assert deterministic-enough outputs (e.g. a date format). Built a prioritized P0 flow list with PMs/feature owners (took persistent calendar-chasing). Chose Playwright (real-browser, mirrors manual QA), wired into Azure DevOps with custom pipeline code to trigger, capture pass/fail, and auto-distribute reports pre-release. Designed for extensibility; documented the "how to add a test" pattern.
**Result**: Automated 10+ previously-manual flows. Caught multiple UI regressions before they shipped. Cut manual QA from 1–2 people × a full day+/release to a hands-off pipeline. A new grad ramped and added tests using my docs.

**Earned Secret**: When the problem is ambiguous, decompose the space *before* writing code. Separating nondeterministic (eval) from deterministic (E2E) made every downstream decision easy.
**Lead with**: the decomposition, not Playwright. **Watch out for**: don't let it collapse into "I set up Playwright." Pairs with GenAI-depth questions (eval / LLM-as-judge boundary).

---

### S002 — Knowing When to Concede (Test Runner Removal) *(Strength 4.0)*

**Situation**: D365 Sales Hub. I'd built an internal test runner that ran API tests against other cluster microservices. During a geo-by-geo rollout it caused intermittent deployment failures blocking other services in the same package. A peer senior engineer recommended removing it from all geos.
**Task**: Decide whether to defend/fix or accept removal — get to the right call for deployment reliability, not the one that protected my work.
**Action**: Joined debugging myself; failure looked transient/geo-specific, a retry cleared it. When it recurred and he escalated, I pushed back — but instead of just arguing, I asked him *why* full removal. That surfaced the real cost: each deploy took 3–4 hrs and any mid-deploy failure forced a full restart. Proposed auto-retry (platform abstracted — couldn't modify retry behavior); proposed scoping to passing geos only (would need two main branches for a subset-coverage, no-customer-exposure service — overhead not worth it). Conceded on the merits, built a clean removal plan with him, documented the rationale so no one re-adds the landmine, and looped in my manager who agreed.
**Result**: Failures stopped; deployments normalized; his workflow safe from multi-hour restarts. Zero customer impact (internal). Coverage loss accepted as reasonable. Applied the lesson on a later rollout with the same engineer — front-loaded a deployment-impact analysis and validated in a test subscription, giving him confidence to greenlight.

**Earned Secret**: I was evaluating a *technical event*; he was living its *operational cost*. The disagreement dissolved when I asked "why" instead of defending "what."
**Lead with**: the question, not the concession. **Watch out for**: don't let it read as deferring to seniority — you evaluated two real alternatives first.

---

### S003 — NSP Security Compliance Deployment *(Strength 4)*

**Situation**: Mid-level on D365 Sales Hub Premium; manager assigned me to lead a company-wide security initiative — associate every cloud resource across 23 global Azure subscriptions with Network Security Perimeters to close a data-exfiltration risk. Hard 2-month deadline, no flex.
**Task**: Design the NSP profile architecture and roll out across 100+ resources / 23 subscriptions with least-privilege rules, breaking nothing.
**Action**: Refused to design until I'd read the codebase to map real traffic per resource type (Key Vaults, storage, Spark-accessed resources — all distinct paths). Rejected both extremes (one profile = too coarse; one-per-resource = unsustainable); designed profiles grouped by traffic pattern. Validated end-to-end in a test subscription, deployed in learning mode, wrote precise rules from live traffic. Daily loop with a senior engineer who held subscription access. Hit an Azure quota block on a few subscriptions — raised requests and parallelized into unblocked geos so it cost zero calendar time. Documented the decision-making and shared it; two other teams picked it up.
**Result**: 100+ resources secured across all 23 subscriptions within the deadline; no legitimate traffic disrupted, no features broken.

**Earned Secret**: Read actual system behavior before making architectural decisions. (Callback: later reused an existing entity-field injection mechanism for an LLM feature instead of building a new path.)
**Lead with**: the investigate-before-design move, not the compliance task. **Watch out for**: frame the senior engineer as a collaborator who held access — not a challenger or a decision-maker over the design. (For an actual disagreement story, use S002.)

---

### S005 — Copilot Usage Analytics Dashboard *(Strength 4)*

**Situation**: Sole engineer on D365 Sales, delivering leadership's first visibility into Copilot skill usage. Three hard things at once: 1.5-month deadline, 1TB/day telemetry, zero prior experience with any system involved.
**Task**: Own it end-to-end — requirements, learn the data platform, build the pipeline, integrate Power BI, ship a daily-updating dashboard.
**Action**: Locked scope with leadership up front (which skills, which metrics). Taught myself an internal big-data platform + SCOPE (SQL-like) — but found a senior engineer who knew both and used their guidance to move faster instead of grinding docs alone. Built a pipeline processing daily telemetry → Azure Blob → Power BI reports embedded in the dashboard. Load-tested the initial-refresh worst case with ~200GB dummy data (sized to largest customer) → found a ~3-hr refresh. Brought concrete data to the Power BI team to drive the fix; kept moving on lower volumes so I wasn't blocked. Posted regular updates in a leadership-visible channel, flagged the refresh risk honestly. Caught myself under-scheduling cross-timezone meetings to protect my own hours, recognized it was the bottleneck, and switched to booking the earliest slot that worked for the other team.
**Result**: Shipped within 1.5 months. Power BI optimizations cut refresh ~3 hrs → ~1 hr, making daily automation practical. Leadership got first-ever day-over-day / month-over-month usage visibility. Later deprioritized (org shift to agentic features), but delivered and handed off successfully.

**Earned Secret**: Cross-timezone coordination is a logistics problem to solve immediately, not a personal inconvenience to minimize. (Secondary: struggling alone is a false economy — knowing *when* to ask is a skill.)
**Watch out for**: Own your real contribution on the refresh — you *found & quantified & drove* it; the Power BI team owns the fix. Don't overstate GenAI depth (analytics *about* Copilot, not building it — S004 is the hands-on LLM story). Lead the reflection with the timezone habit — it's a genuine "something I'd do differently" beat.

---

### S006 — Configurable Validations Framework (Rippling) *(Strength 4 — Customer Obsession story)*

**Situation**: Junior engineer at Rippling on Unified Churn — the product handling a customer offboarding from a Rippling product. A customer representative raised the churn request and followed it through. There was **no validation at request time**: the system accepted every request. The eligibility criteria (pending dues and similar) were only checked deep inside the processing pipeline, so a request that failed them silently entered a stuck in-progress state. A normal churn completed in about a day; the rep would wait 3+ days before concluding something was wrong and filing a support ticket — **and while it sat there, the customer was still being billed for a product they'd asked to leave.** Then an engineer (usually me or my mentor) had to debug the request by hand, work out which criterion had failed, and hand-write the explanation into the ticket. Every single occurrence. Volume was low — under 10 churns a month — which is exactly why it had been sitting in the backlog unprioritized.

**Task**: A senior engineer assigned me to fix it. I was the person on the receiving end of those tickets, so I knew the shape of the pain first-hand.

**Action**: The logs on stuck requests pointed at failing validations as the cause — and the real defect wasn't that the failure was reported badly, it was that a request which couldn't possibly succeed had been **accepted in the first place**. So rather than adding alerting or better error surfacing on the stuck state, I moved validation to the point of entry: an API endpoint called on churn form submission that ran the checks and returned failures to the rep before the request was ever created. Rather than writing a second copy of the criteria at the endpoint, I **refactored the existing pipeline validation logic out into a registry** — a dict keyed by the product being churned, mapping to the list of validator functions to run; each validator took the product-level and company-level churn request and returned a boolean plus an error string. One source of truth, so entry-point and pipeline checks could never drift. Pipeline validation stayed in place as a second line of defence. My mentor oriented me on the existing system and, on design review, suggested adding the ability to override specific validations or downgrade their severity — so hard blockers stayed non-negotiable while others could be tuned from the internal admin UI without a deploy, with overrides persisted as validation-rule DB records. I also cleaned up the existing victims: the already-stuck requests were cancelled, with care taken that a mid-churn cancellation didn't leave the customer in a broken state.

**Result**: The rep got the blocking reason at the point of submission, could fix it, and watch it pass. The entire class of "accepted request that silently dies days later" disappeared — no churn request could be created that didn't satisfy its validations, so there was nothing left to debug by hand, and no more days of a customer being billed for a product they were trying to leave. No stuck-request tickets came in afterwards. The framework stayed in active use after I moved on.

**Earned Secret**: A silent failure buried deep in a pipeline isn't a reporting problem, it's a **placement** problem. Move the check to where the person who can fix it will see it. Corollary: when you move a check, refactor the existing logic into one shared place — don't leave a second copy behind to drift.

**Lead with**: the customer still being billed while the request sat stuck. Not the ticket toil, and not the registry design — the person who couldn't find out what was wrong and was paying for the privilege.

**What I'd do differently** *(genuine — you volunteered it under drilling)*: I inferred the customer cost entirely from the tickets I personally debugged. I never went and talked to the support or CS people who fielded them. Had I done that, I'd have understood the billing consequence sooner and could have made a much stronger case for pulling it out of the backlog earlier. Secondary: I shipped it and never instrumented the outcome — I can tell you the failure class was eliminated by construction, but I never measured whether the severity-override knob was actually used, which means I can't say whether that part earned its complexity.

**Watch out for**:
- **Don't inflate the scale.** Under 10 churns/month. If asked volume, say it plainly and add *why that mattered*: low volume is precisely why it stayed in the backlog and stayed easy to defer. That's the better answer anyway — you fixed unglamorous work nobody would have blamed you for ignoring, and the per-incident cost was high even if the count was low.
- **Quote no percentage.** The old resume line said "20% fewer tickets"; it was an unmeasured estimate and it has been removed from the resume. Never reintroduce it. Impact here is *mechanism* — a class of failure eliminated by construction — not a measured delta.
- **Get attribution exactly right.** The **refactor-instead-of-duplicate call was yours** — that's the best technical decision in the story, own it. The **override / severity-downgrade capability was your mentor's suggestion at design review** — credit them. Getting this backwards in either direction is the trust risk; volunteering the mentor's contribution unprompted actually reads as Earn Trust.
- **Do not claim the multi-page validation was a flaw.** Validation surfaced per page in a multi-step churn flow — that's contextual validation, and it's correct. (An earlier draft of this file wrongly framed it as serial-discovery friction. It isn't.)
- **Oldest story, weakest recall.** If drilled past what you actually remember: *"This was about four years ago and it's the work I remember least precisely — the shape was X, but I'd rather tell you what I'm confident about than reconstruct details."* Survivable. Inventing architecture is not.
- **Unknowns, answer honestly**: whether anyone added validators after you left, and whether the severity config was ever exercised — you don't know. "I don't know, I'd left by then" costs nothing. Guessing costs the question.

---

### S007 — Payroll Filing-Fee Automation (Rippling) *(Strength 4.5 — your one bulletproof number)*

**Situation**: Junior engineer at Rippling. When a customer offboarded from Rippling's payroll product, if they met certain additional criteria they were liable for a final, flat **$1,000 filing fee** — payment for filing work Rippling still had to do on their way out. Collecting it was a manual step: the ops team was supposed to notice an eligible churn and reach out to the customer. Under high volume of incoming churn requests, that step just got forgotten. My mentor raised this in conversation as something he suspected was costing us money. Nobody had quantified it.

**Task**: Find out how big it actually was, and if it was worth fixing, fix it. Nobody assigned this — it came out of a conversation.

**Action**: I went and queried the database directly rather than estimating: customers with existing churn requests who met the fee eligibility criteria and had never been charged. **45 customers, $1,000 each — $45,000 uncollected.** I brought that number to my mentor. The number is what turned an offhand remark into work, and it also exposed urgency: once a customer had *fully* churned out there was no billing relationship left and the money was gone permanently. Only customers still mid-churn were recoverable, so every week of delay converted recoverable revenue into unrecoverable. Two pieces of work: (1) **Stop the leak** — the churn process was an automated sequence of steps, so I added a step that generated the filing-fee invoice automatically for any eligible payroll churn, removing the human from the loop entirely. (2) **Recover what was still recoverable** — backdated invoices for the 45 customers still in progress. Since this was real customer money, I got PM sign-off on retroactively invoicing them rather than making that call myself. On safety: invoice creation was idempotent, and actual collection was handled by separate billing infrastructure with its own idempotency, so a re-run couldn't double-charge. Deliberately, my code only *generated* an invoice — the customer still paid it themselves; nothing auto-charged a card. I also built the admin-facing breakdown of the filing-fee charge on the invoice, so the amount wasn't an unexplained line item.

**Result**: $45,000 in backdated invoices generated and charged to the 45 recoverable customers. The manual step was eliminated — eligible churns now generate the fee invoice as part of the automated churn sequence, so a high-volume week can't cause missed revenue anymore. Nothing broke in production. Still in use when I left.

**Earned Secret**: An offhand remark becomes a priority the moment someone puts a number on it. The highest-leverage thing I did wasn't the code — it was spending an afternoon on queries to turn "I think we're losing money here" into "45 customers, $45,000, and it's shrinking every week." Also: when a leak is time-sensitive, sizing it *is* urgency work, not analysis paralysis.

**Lead with**: the number, and the fact that you went and got it. **Never lead with** "I automated a manual process."

**Watch out for**:
- **This is NOT a Think Big or "found a problem nobody asked about" story.** Your mentor surfaced it. State that plainly and early — *"my mentor raised it, I quantified it"* — and the story loses nothing, because the quantification is the impressive part and it's entirely yours. Claiming discovery gets you caught for no upside.
- **Derive the $45K, don't recite it.** 45 eligible customers × $1,000 flat = $45,000, from queries you ran yourself. If asked, walk the arithmetic. This is the one number in your entire set that survives arbitrary drilling — use it, and let it do the work the $156K currently can't.
- **Be precise on "recovered."** $45K is a one-time recovery of backdated invoices, **not** an annual figure. And be honest that customers who had already fully churned were *unrecoverable* — that money was permanently lost. Volunteering that is stronger than being caught by "so you got all of it back?"
- **You didn't charge anyone automatically.** Your code generated an invoice; the customer paid it. If an interviewer starts probing "you retroactively charged customers?", the answer is invoices with PM sign-off, no automatic card charges.
- **No failure beat exists here.** Nothing broke. If pushed for what went wrong, don't invent — pivot to the honest limitation (the unrecoverable cohort) or the safety design you'd have to defend if it *had* gone wrong.
- **Don't oversell the UI.** It was the invoice breakdown for the filing-fee charge, not an admin console. The resume says "admin-facing UI" — keep the spoken version accurate to what it was.

---

## Tell Me About Yourself

> **Structure:** present → path → why here. ~60–75 seconds spoken. Names outcomes, not
> narratives — the numbers ($156K, 23 subscriptions, LLM eval-vs-E2E) are bait so the interviewer
> *pulls* on S001 / S003 / S004 rather than you spending the story unprompted. Front-loads the two
> lead stories + the under-used S003, and closes the "why leave / why US" beat proactively.

### Primary version
> "Sure. I'm a software engineer with about three and a half years of experience, most recently at
> Microsoft on the Dynamics 365 Sales team, and before that at Rippling. I did my undergrad at IIT BHU.
>
> The through-line of my work has been **owning backend and platform problems end-to-end** — usually
> the kind that start out ambiguous or high-stakes and need someone to impose structure on them. At
> Microsoft I worked deep in the D365 Sales platform: I ran a live, zero-downtime migration of our
> config store off Azure Table Storage onto Cosmos DB — which saved around $156K a year and shrank our
> fault boundary from a whole geo down to a single island. I also led a company-wide security rollout
> across 23 Azure subscriptions on a hard two-month deadline, and I built the end-to-end
> regression-testing system for our LLM-based sales agent — which was interesting because it meant
> drawing a real line between what you test deterministically and what needs a separate eval system for
> the model's output.
>
> That last one is where a lot of my recent interest sits — I've been doing hands-on GenAI work, both
> at Microsoft and on a side project, an MCP-based documentation assistant using local embeddings and
> retrieval.
>
> Earlier, at Rippling, I was closer to the product surface — payroll and offboarding — where I got
> good at spotting where the business was quietly leaking money or engineering time and fixing the
> whole class of problem, not just the ticket.
>
> I'm a US citizen and my family's here, so after building a solid foundation across those two
> companies, moving to the US now is the deliberate next step — and I'm looking for a role where I can
> keep owning meaty platform and backend problems at a larger scale."

### Delivery notes
- **Backend/infra-heavy role** → lead the middle paragraph with the migration (S001); drop or shorten the GenAI line.
- **AI/GenAI role** → flip it: lead with the LLM regression system + side project, migration second.
- Keep the Rippling paragraph short — texture, not a headline. Cut it entirely if running long.
- Ends on the "why US" beat in your own framing, so it's a deliberate story — not a defensive answer pulled out of you later.

---

## Gap-Handling Scripts

> **Disagreement / conflict is no longer a gap.** You now have S002 (peer disagreement, changed your mind, conceded) and S003 (senior challenged a security decision, defended with due diligence). Use S002 as the primary "disagreement / conflict" story and S003 as the backup. The old "I haven't had a real disagreement" script is retired — don't use it.

### Giving Feedback *(still a genuine gap)*
> "I haven't been in a formal position to give feedback yet — most of mine has been peer code-review comments. My principle is specific and actionable over vague: I'd rather say 'this function will fail on null input — here's the fix' than 'the code needs improvement.'"

---

### Receiving Feedback
Deploy **S001** (admitted fault; manager gave direct corrective guidance on recovery; accepted without defensiveness, acted immediately) or **S002** (peer's feedback led you to remove your own service once you understood the operational cost). Both show accepting input and acting on it.

---

### Prioritization Under Competing Demands
> "The clearest example is my config migration incident at Microsoft. When I found data missing mid-rollout, four things felt urgent at once: fix the code, resume migration, recover the affected geo, and tell my manager. It got simple once I asked 'what's the blast radius of waiting on each?' — stop new damage first (flag off / pause), recover existing damage second (cross-geo copy), fix root cause third (the abstraction + PR), validate nothing else broke fourth (write-path audit). I sequence by blast radius, not effort."

If pushed for project-level prioritization: *"At my stage I've mostly executed on a defined backlog — my judgment has been sequencing within a task, not between competing projects. That's something I'm looking forward to developing at the next level."*

---

### "Why did you leave Microsoft?"
> "I'm a US citizen and my family is based here. After 3.5 years across Rippling and Microsoft, I had enough foundation to make the move to the US at the right time — personally and professionally. A deliberate decision, not a reactive one."

Follow-up "Why now?": *"My mother has been in the US for a while. I wanted to wait until I had enough experience to be competitive here — mid-level with Microsoft and Rippling on the resume felt like the right moment."*

---

## Notes on Deploying These

- **Ambiguity** shows up three ways — S004 (no spec → decompose), S005 (unknown tech + requirements gathering), S003 (design not obvious → investigate first). Pick by what else the question is probing.
- **GenAI** — S004 is your hands-on LLM story (eval vs. E2E). S005 is GenAI-adjacent. If asked for hands-on model work, also have your MCP/RAG documentation-assistant side project ready as a separate talking point (it's not in the numbered set but it's real).
- **Self-awareness / "a weakness you fixed"** — S005's timezone habit is the cleanest; a real behavior you changed with a concrete before/after.
