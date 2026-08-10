# Amazon Leadership Principles — Loop Operating Manual
**Aayush Alok** · Last updated: 2026-08-10

> **What this file is.** The Amazon-specific routing layer. Story *content* lives in `behavioral_prep.md` and the `S00x_*_prep.md` depth files — this file never repeats it. This is: what each LP is actually scoring, which story to route to it, the sentence to open with, and how to survive a full loop without burning a story three times.
>
> **Read this the morning of the loop. Read `behavioral_prep.md` the week before.**

---

## 1. How the loop actually works

- **4–5 interviewers**, each assigned **2–3 LPs**. That's **10–14 LP probes** across the day.
- They are scoring **you against LPs**, not against topics. The question is a vehicle; the LP is the rubric.
- **They debrief together.** Everything you say is written down and compared. A story told to three interviewers is visible in a way it is never visible inside a single round.
- One interviewer is the **Bar Raiser** — outside the hiring team, no stake in filling the role, and the one who drills hardest. Usually the one who says *"walk me through what **you** specifically did."*
- Expect **STAR-shaped follow-ups**: what was the situation, what was *your* task, what did *you* do, what was the measurable result, what would you do differently.

**Your bench: 8 stories against 10–14 probes.** That is thin but workable at a **2-use cap per story**. It only works if you route deliberately.

---

## 2. Story budget — allocate before you walk in

| Story | Strength | Max uses | Spend on |
|---|---|---|---|
| **S001** Config Migration | 4.5 | 2 | Earn Trust · Ownership · Bias for Action · Broad Responsibility |
| **S004** LLM Regression Testing | 4.5 | 2 | Ownership · Invent & Simplify · Highest Standards · Customer Obsession *(backup)* |
| **S007** Filing-Fee Automation | 4.5 | 2 | Frugality · Dive Deep · Earn Trust |
| **S002** Test Runner Removal | 4.0 | 2 | **Backbone (only coverage)** · Learn & Be Curious · Earn Trust |
| **S003** NSP Rollout | 4.0 | 2 | Are Right A Lot · Deliver Results · Think Big |
| **S005** Copilot Dashboard | 4.0 | 2 | Learn & Be Curious · Dive Deep · Earth's Best Employer |
| **S006** Configurable Validations | 4.0 | 2 | **Customer Obsession (primary)** · Invent & Simplify |
| **S008** BCDR Drill | 3.0 | 1–2 | **Cap relief only** — Bias for Action / Highest Standards when S001 or S003 is spent |

**Running tally.** Keep a mental count between rounds. After each interview, note which stories you spent. If S001 has carried two probes by lunch, **it is retired for the day** — route to S006/S007/S008 even when S001 fits better. Three interviewers hearing the config migration debrief you as a one-story candidate, and that impression survives everything else you said.

---

## 3. The 16 LPs — what each one is actually scoring

> The left column is the thing most candidates get wrong. Misreading what an LP scores is how you tell a true, well-delivered story and still lose the question. *(You have already done this once — see the do-not-use list in `behavioral_prep.md`.)*

### Tier 1 — near-certain to appear

**1. Customer Obsession** — *Scores: did the customer **drive a decision you made**, not did your work benefit customers.*
- **Primary: S006.** Open on the person: *"A customer asked to leave a product, was told the request went through, and then sat stuck for days while still being billed for it."*
- **Backup: S004.** Only if you lead on the customer-driven decision: *"We built the P0 flow list with PMs by **customer criticality**, not by code coverage — and I went to the QA vendors first to learn the flows customers actually walk instead of assuming."* Then say the proxy out loud: *"my customer signal was the vendors and the PMs — I wasn't talking to customers directly."*
- **Trap:** on S004, drifting into eval-vs-E2E. That's Invent & Simplify and it will be scored as such. One sentence of context, no more.
- **Honest ceiling:** S003/S005/S002 are second-order. If forced there, say "my customer here was internal" — it scores better than an inflated claim.

**2. Ownership** — *Scores: did you act beyond your assigned box, and did you stay with it through the ugly part.*
- **"Owned it end-to-end": S004** (primary), S003, S005.
- **"Stayed with the ugly part": S001** — *"I caused production data loss in my own migration, and nobody took it off my hands."*
- ⚠️ **"Something nobody asked you to do" is a GAP.** S007 does not qualify — your mentor surfaced it. Do not claim discovery. Use the bridge, then give S001 (owned the recovery) or S008 (went past the documented minimum).

**3. Invent and Simplify** — *Scores: did you collapse a hard problem into a simpler one, or reduce the number of moving parts.*
- **Primary: S004** — *"You can't write assertions against a nondeterministic system. One cut made it tractable: output **quality** to an eval system, the **deterministic** layer — setup, rendering, access — to E2E. After that, every remaining decision was obvious."*
- **Backups:** S006 (one registry, not a second copy of the checks), S003 (profiles by traffic pattern — rejected both extremes), S001 (`ConfigStore` — one routing point).

**4. Dive Deep** — *Scores: did you go and get the actual data instead of reasoning from assumption.*
- **Primary: S005** — the ~200GB load test sized to the largest customer, tracing ~3hrs to the Power BI layer and not your pipeline.
- **Backup: S007** — queried the DB rather than estimating. **S003** — read the codebase to map real traffic. **S001** — the write-path audit from a checklist, not memory.
- **Trap:** Dive Deep is not "I worked hard." It's "I distrusted the summary and went to the source."

**5. Bias for Action** — *Scores: did you move under uncertainty, with reversibility considered.*
- **Primary: S001** — flag off within 15 minutes, escalated before having a fix.
- **Backups:** S003 (learning mode rather than waiting for perfect knowledge), S007 (the recovery window shrinking every week), **S008** (ran the external dependency a week early).

**6. Earn Trust** — *Scores: do you tell the truth when it costs you, and give credit away.* **Your strongest LP.**
- **Primary: S001** — admitted fault to your manager *before* you had a fix.
- **Backups:** **S002** (built the removal plan *with* the person you'd argued against, and documented it so it couldn't be quietly undone), **S005** (raised the refresh risk while it was still a risk, with no fix in hand), **S007** (PM sign-off before retroactively invoicing; credited your mentor unprompted), **S006** (the override capability was your mentor's idea — say so).

**7. Deliver Results** — *Scores: did you hit the commitment, and what did you do when it got hard.*
- **Primary: S003** — 100+ resources, 23 subscriptions, hard 2-month deadline, quota block absorbed by parallelizing into unblocked geos at zero calendar cost.
- **Backups:** S005 (shipped in 1.5 months, zero prior experience), S007.
- ⚠️ **"A time you missed / didn't deliver" is a GAP.** Nearest true thing: S002 — you committed a test runner into the shared pipeline and had to retract the coverage when you couldn't keep it disruption-free. Stay on the commitment you dropped, not the peer dynamic.

**8. Insist on the Highest Standards** — *Scores: did you refuse the acceptable-but-worse option when nobody would have caught you.*
- **Primary: S004.**
- **Backups:** S001 (audited every write path from a checklist rather than memory), S006 (kept pipeline validation as a second line of defence rather than removing it once entry-point checks existed), **S008** (refused the documented one-day buffer).

**9. Have Backbone; Disagree and Commit** — *Scores: did you hold a position on **the merits of a decision**, at cost, past the first "no" — and then own the outcome you argued against.*
- **S002 Version B is your only coverage.** Front-load the resistance; make the commit the closer. Four beats in order — held ground · made him justify it · did the work before yielding · committed all the way. Full decision table in `S002_test_runner_removal_prep.md`.
- ⛔ **Never route your own-workload memory here.** Backbone is never about your capacity or defending your own work from removal. See the do-not-use list.
- **Backup: S003** — challenged a security decision, defended with due diligence.

### Tier 2 — likely

**10. Learn and Be Curious** — *Scores: did you close a knowledge gap on purpose.*
- **Primary: S005** — internal big-data platform + SCOPE from zero, in 1.5 months, and found a senior engineer instead of grinding docs alone.
- **Backups:** MCP/RAG side project (local embeddings, retrieval), S002 (asked *why* instead of defending *what*).

**11. Are Right, A Lot** — *Scores: quality of judgment under incomplete information, and whether you sought disconfirming input.*
- **Primary: S003** — *"I couldn't learn every traffic pattern from the code, so rather than guess at access rules I deployed in learning mode and wrote them from observed live traffic."* Committing while still not knowing.
- **Backup: S002** — updated your position on evidence you demanded and verified yourself.
- **Collision rule:** *decision under uncertainty* → **S003**. Never S004 — ambiguity ≠ uncertainty; you resolved S004 by asking.

**12. Frugality** — *Scores: more output per unit of resource. Money **or** engineering time.*
- **Primary: S007** — derive it live: 45 customers × $1,000 flat = **$45,000**. Your one number that survives arbitrary drilling.
- **Backups:** **S004** (1–2 people × a full day, every release, eliminated permanently), S001 ($156K — **attribute to your manager, say "forecast," then pivot to the mechanism**: per-geo dedicated resources decommissioned onto already-funded shared infrastructure).

**13. Think Big** — *Scores: did you propose scope beyond what you were handed. At 3.5 years the bar is not "redirected strategy."*
- **Primary: S003** — company-wide, 23 subscriptions, and two other teams adopted your documented approach.
- **Backup: S007** — nobody had sized the leak; quantifying it created the workstream.
- **Trap:** do not inflate these into moonshots. An honest mid-level answer beats a stretched one, and *"I don't have one"* scores worse than either.

### Tier 3 — rarely probed for SDE, but have an answer

**14. Hire and Develop the Best** — ⚠️ **Genuine gap — bridge, never dead-end.**
- Say: *"I haven't been in a formal mentoring or hiring position yet. The closest is enabling rather than coaching — I documented the pattern for adding tests to the LLM agent, and a new grad used that to ramp up and start contributing tests independently."* Then the feedback principle: specific and actionable over vague — *"this will fail on null input, here's the fix"* rather than *"this needs improvement."*
- **Do not claim mentoring you haven't done.** There is no beat in any of your eight stories where you developed a person; relabeling won't create one.

**15. Strive to be Earth's Best Employer** — *Scores: did you make work better for the people around you, at cost to yourself.*
- **S005** — the timezone habit. You were under-scheduling cross-timezone meetings to protect your own hours, recognized you'd become the bottleneck, and switched to booking the earliest slot that worked for the other team. A real behavior change with a before and after.

**16. Success and Scale Bring Broad Responsibility** — *Scores: did you think about who gets hurt when it goes wrong.*
- **S003** (exfiltration risk on customer data; refused to guess at rules because a wrong rule breaks live customer traffic), **S001** (blast radius geo → island — capping how many customers one failure can ever reach), **S007** (billing fairness: took retroactive invoicing to PMs rather than deciding unilaterally, and built the breakdown so the charge wasn't unexplained).

---

## 4. Question stems by LP — what you'll actually hear

> **How to drill this.** Cover the right-hand side. Have someone read stems at random; say **only the story ID and the first sentence** — not the story. 10 seconds each. Picking the story is a separate skill from telling it, and it's the one that fails live.
>
> ⚠️ marks a stem where you have **no clean answer**. Do not discover these in the room — the bridge is written inline below, and the longer scripts are in **Gap-Handling Scripts** in `behavioral_prep.md`.

### 1. Customer Obsession
- Tell me about a time you went above and beyond for a customer → **S006**
- How do you know what your customers actually want? → **S006** *(the tickets were the signal)* + the honest limit: you never talked to support/CS
- Tell me about a time you used customer feedback to improve something → **S006**
- Tell me about a time you had to balance customer needs against business needs → **S007** *(fairness beat: PM sign-off, invoice breakdown)*
- Tell me about a time you prioritized based on customer impact → **S004** *(P0 list by customer criticality, not code coverage)*
- ⚠️ Tell me about the most difficult customer you've dealt with → **no direct answer.** Bridge: your customer contact was through reps and tickets, then give S006.
- ⚠️ Tell me about a time you couldn't meet a customer's expectation → nearest true thing is S006's *stuck requests you cancelled* — the customers whose requests you killed. Say it plainly.

### 2. Ownership
- Tell me about a project you owned end to end → **S004** *(or S005, S003)*
- Tell me about a time you had to work on something you had no expertise in → **S005**
- Tell me about a time you took on a problem and stayed with it through the hard part → **S001** *(caused it, recovered it)*
- Tell me about a time you sacrificed short-term speed for a long-term outcome → **S003** *(refused to design until you'd read the system)* or **S001** *(`ConfigStore` before resuming)*
- ⚠️ **Tell me about a time you took on something outside your job responsibilities** → **your most-likely-asked gap.** Do NOT use S007 (mentor surfaced it). Bridge, then **S008** (went past the documented minimum) or **S001** (owned the recovery).
- ⚠️ Tell me about something you did that nobody asked you to do → same gap, same bridge.

### 3. Invent and Simplify
- Tell me about a time you simplified something complex → **S004** *(the deterministic cut)*
- Tell me about your most innovative solution → **S004**, backup **S006** *(registry)*
- Tell me about a time you found a simple solution to a hard problem → **S003** *(rejected both extremes, grouped by traffic pattern)*
- Tell me about a time you took a different approach than everyone expected → **S006** *(moved the check to the entry point instead of improving the error reporting)*
- Tell me about a time you eliminated work rather than optimizing it → **S006** or **S007**

### 4. Are Right, A Lot
- **Tell me about a decision you made with incomplete information** → **S003** *(learning mode instead of guessing). Your cleanest answer in the whole set.*
- Tell me about a time you changed your mind based on new information → **S002 Version A**
- Tell me about a time you were wrong → **S001** *(the missed write path)* or **S002 Version A**
- Tell me about a time your judgment was questioned → **S002** *(and you demanded the evidence before yielding)*
- Tell me about a bad decision you made → **S001**
- How do you make good decisions fast? → S001 blast-radius sequencing as the framework

### 5. Learn and Be Curious
- Tell me about a time you had to learn something new to finish a project → **S005**
- What's the most interesting thing you've learned recently? → **MCP/RAG side project** *(local embeddings, retrieval)*
- Tell me about stepping outside your comfort zone → **S005** *(three unknowns at once)*
- Tell me about a time curiosity changed an outcome → **S002** *(asked* why *instead of defending* what*)*
- How do you stay current? → side project + the eval-vs-E2E boundary work

### 6. Hire and Develop the Best ⚠️ **entire LP is a bridge**
- ⚠️ Tell me about a time you mentored someone
- ⚠️ Tell me about giving someone difficult feedback
- ⚠️ Tell me about helping a struggling teammate
- ⚠️ How do you develop the people around you?
- **All of the above → the bridge in §8, then:** the new grad who ramped and contributed tests off your documentation, plus the code-review principle *(specific and actionable over vague)*. **Do not claim mentoring you haven't done.**
- Tell me about someone who helped *you* grow → **S005** *(the senior engineer on the data platform)* or **S006/S007** *(your mentor)* — this one you can answer well, and it's the version most often asked of mid-level candidates.

### 7. Insist on the Highest Standards
- Tell me about a time you weren't satisfied with the status quo → **S006**
- Tell me about a time you refused to lower the bar under pressure → **S008** *(refused the documented one-day buffer)*
- Tell me about the highest-quality work you've produced → **S004**
- Tell me about a time you set a standard others followed → **S003** *(two teams adopted the doc)*, **S004** *(the add-a-test pattern)*
- Tell me about a quality tradeoff you had to make → **S002** *(accepted real coverage loss for deployment reliability)*
- Tell me about a time you caught something others missed → **S001** *(write-path audit)*, **S008**

### 8. Think Big
- Tell me about the most ambitious thing you've worked on → **S003**
- Tell me about a time you proposed something beyond your assignment → **S007** *(sizing created the workstream)*
- Tell me about work of yours that had impact beyond your team → **S003** *(two teams picked up the approach)*
- ⚠️ Tell me about a vision you set / how do you think about long-term direction → **level-calibrated.** Give the honest mid-level answer: scope past what you were handed, not strategy. Never *"I don't have one."*

### 9. Bias for Action
- Tell me about a time you had to act fast → **S001** *(flag off in 15 minutes)*
- Tell me about a calculated risk you took → **S003** *(learning mode)*, **S001** *(live migration)*
- Tell me about a time you acted without full data → **S003**
- Tell me about a time speed mattered more than perfection → **S007** *(the recovery window shrinking weekly)*
- Tell me about a time being proactive prevented a problem → **S008**
- Tell me about a two-way door decision → **S001** *(per-geo feature flag; and the honest lesson — deleting before verifying made it one-way)*

### 10. Frugality
- Tell me about a time you saved the company money → **S007** *(derive 45 × $1,000 live)*
- Tell me about doing more with less → **S004** *(1–2 person-days per release, permanently)*
- Tell me about working under resource constraints → **S003** *(Azure quota block — parallelized, zero calendar cost)*
- Tell me about reducing cost in a system → **S001** *(attribute the $156K, then give the mechanism)*

### 11. Earn Trust — **your strongest LP, expect two probes**
- Tell me about a time you admitted a mistake → **S001** *(told your manager before you had a fix)*
- Tell me about receiving difficult feedback → **S001** *(primary)*, **S002** *(backup)*
- Tell me about delivering bad news → **S001**
- Tell me about raising a risk before you had a solution → **S005** *(the ~3hr refresh, in the leadership channel)*
- Tell me about a time you gave someone else credit → **S006/S007** *(your mentor — volunteer it unprompted)*
- Tell me about repairing a relationship after conflict → **S002** *(built the removal plan with him; front-loaded the analysis next time)*
- Tell me about earning trust with a skeptical stakeholder → **S003** *(daily loop with the engineer who held access)*

### 12. Dive Deep
- Tell me about using data to solve a problem → **S007** *(queried the DB rather than estimating)*
- Tell me about the most complex thing you've debugged → **S005** *(~200GB load test → traced to Power BI, not your pipeline)*
- Tell me about finding a root cause others missed → **S005**, **S001**
- Tell me about a time the data contradicted what everyone assumed → **S007** *("probably losing money" → 45 customers, $45,000)*
- Tell me about auditing something rather than trusting it → **S001** *(every write path, from a checklist)*, **S003** *(read the codebase)*
- How do you know your system is healthy? → **S008** *(you don't until you test the failure path)*

### 13. Have Backbone; Disagree and Commit
- Tell me about a time you disagreed and had to commit anyway → **S002 Version B**
- Tell me about the strongest disagreement you've had at work → **S002 Version B**
- Tell me about pushing back on a senior person → **S002 Version B**
- Tell me about a time you convinced someone → **S003** *(challenged the security decision)*
- ⚠️ **Tell me about a time you disagreed with your manager** → **no story.** Answer honestly: your disagreements have been with peers and senior engineers, then give S002 Version B in full. Do not stretch S003, and **never** route your own-workload memory here (see the do-not-use list).
- ⚠️ Tell me about a time you were overruled → thin. S002 is the closest, and you were persuaded rather than overruled — say the difference out loud.

### 14. Deliver Results
- Tell me about meeting an aggressive deadline → **S003** *(2 months, no flex)*, **S005** *(1.5 months from zero)*
- Tell me about overcoming a blocker to deliver → **S003** *(quota block)*
- Tell me about the most challenging project you've delivered → **S003**
- Tell me about a time you had to deprioritize something → **S005** *(kept moving on lower volumes while blocked)*
- ⚠️ **Tell me about a time you missed a deadline / didn't deliver** → **gap.** Nearest true thing: **S002** — you committed a test runner into the shared pipeline and had to retract the coverage when you couldn't keep it disruption-free. Stay on the commitment you dropped, not the peer dynamic.

### 15. Strive to be Earth's Best Employer
- Tell me about improving things for your team → **S005** *(timezone habit)*
- Tell me about a time you took on inconvenience so someone else didn't have to → **S005**
- Tell me about supporting a teammate → **S002** *(his multi-hour deploy restarts were the reason you yielded)*

### 16. Success and Scale Bring Broad Responsibility
- Tell me about considering second-order consequences → **S001** *(blast radius geo → island)*
- Tell me about a decision with impact beyond your team → **S003**
- Tell me about weighing security, privacy, or fairness → **S003** *(exfiltration risk)*, **S007** *(billing fairness → PM sign-off)*
- Tell me about a time you slowed down because the downside was large → **S003** *(refused to guess at rules — a wrong rule breaks live customer traffic)*

---

## 5. Live routing — the 3-second decision

When the question lands, run this in order. Do not classify the LP; classify the **shape**.

1. **Is a person the subject of the question?** *(disagreement, feedback, someone stuck, a customer)* → S002 / S006 / S005. **Not** S003 or S004.
2. **Is a mistake or failure the subject?** → S001. Always S001.
3. **Is a number or business impact the subject?** → S007.
4. **Is "you didn't know what to do" the subject?** → ambiguity → S004 · uncertainty → S003 · unfamiliar tech → S005.
5. **None of the above?** → the story with uses remaining that has the strongest verifiable detail.

**Buy the three seconds out loud:** *"Let me think for a second about the sharpest example."* Silence is recoverable. "I don't have one" is not.

---

## 6. "Tell me another one" — the most common failure mode

Amazon interviewers routinely ask for a **second example of the same LP** in the same round. This is where thin benches collapse. Pre-decide the pairs:

| LP | First | Second |
|---|---|---|
| Customer Obsession | S006 | S004 *(lead on the customer-criticality prioritization)* |
| Ownership | S004 | S001 *(the recovery)* |
| Bias for Action | S001 | S008 |
| Highest Standards | S004 | S008 or S001 *(the write-path audit)* |
| Earn Trust | S001 | S002 *(the repair)* or S007 *(PM sign-off)* |
| Dive Deep | S005 | S007 |
| Invent & Simplify | S004 | S006 |
| Deliver Results | S003 | S005 |
| Backbone | S002 Version B | S003 *(challenged the security decision)* — then stop; say honestly that's your set |

---

## 7. Follow-up survival

**"I" not "we."** Amazon penalizes collective voice harder than any other company. Every time you say "we decided," an interviewer writes down *unclear contribution*. Say what the team did in one clause, then **"my part was ___"**. Rehearse this on S003 and S005 specifically — both involve a senior engineer, and both risk sounding like you assisted.

**Bring the data unprompted.** Every story should surface at least one number before you're asked for one.

**Numbers you can defend, in order of durability:**

| Number | Durability | Rule |
|---|---|---|
| **$45,000** = 45 customers × $1,000 flat | **Bulletproof** — you ran the queries | Derive it live, don't recite it. One-time recovery, **not** annual. Volunteer that the fully-churned cohort was unrecoverable. |
| 23 subscriptions / 100+ resources / 2 months | Solid | Yours end-to-end. |
| 10+ automated flows; 1–2 person-days per release | Solid | Effort reduction, not a percentage. |
| ~1TB/day telemetry; ~200GB load test; ~3hr → ~1hr | Solid | You found and drove it; **Power BI owned the fix** — say so. |
| <10 churns/month (S006) | Solid, and say it plainly | Low volume is *why* it stayed in the backlog. That's the better answer. |
| 1-day documented buffer vs 3-day fix (S008) | Solid | The arithmetic is the story. |
| **$156,000/yr** | ⚠️ **Fragile** | Your manager's COGS **forecast**, not your calculation. Attribute unprompted, then pivot to the mechanism. |
| ~~"20% fewer tickets"~~ | ⛔ **Deleted** | Was never measured. Removed from the resume. Never reintroduce it. |

**"I don't know" is a complete answer** when it's true — *"I don't know, I'd left by then"* costs nothing. Guessing costs the question, and the Bar Raiser is specifically testing whether you'll bluff.

**Always have the "what would you do differently."** Amazon asks it on nearly every story. Pre-loaded: S006 (never talked to support/CS; never instrumented whether the override knob was used) · S008 (didn't fix the timing in the runbook) · S005 (the timezone habit) · S001 (test env didn't replicate the low-frequency write path — own the gap).

---

## 8. Banned in the room

> *"I do not have any such experience."* · *"I've never had such a moment."* · *"Not sure how to answer."* · *"Hypothetically, I would…"* · **"Luckily, there was no customer impact."** *(containment was your design — say "contained by design," then say how)* · **"overwhelmed" / "bandwidth"** *(reads as low Ownership regardless of context)*

---

## 9. Morning-of checklist

1. Read §3 headers only — the *scores:* line for each LP. Nothing else.
2. Say the seven **customer-forward opening sentences** out loud (table in `behavioral_prep.md`). 10 seconds each.
3. Derive **$45K** out loud: 45 × $1,000.
4. Say the **$156K attribution sentence** out loud once, in full.
5. Say the **S008 arithmetic** out loud: runbook said one day, the fix took three.
6. Reread the **do-not-use list** in `behavioral_prep.md`. Those are true memories that fit the shape of real questions — that's exactly why they surface under pressure.
7. Decide your **opening story** for a free choice: S001 or S004.
