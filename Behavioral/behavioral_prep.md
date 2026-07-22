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
| Business / cost impact | S001 ($156K/yr), S004 (eliminated 1–2 person-days/release) |
| Above and beyond for a customer | Rippling Configurable Validations |
| Couldn't meet a commitment | S002 |
| Digging into details to understand a complex problem | S005 (perf investigation), S003, S001 |
| Giving feedback | Script (gap) |
| "Why did you leave Microsoft?" | Script |

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

### S002 — Test Runner Removal
**Opener:** *"Someone recommended deleting a service I'd built. I disagreed — and what resolved it wasn't an argument, it was a question I asked him."*
**Closer:** *"I was evaluating a technical event; he was living its operational cost. Understand the operational constraints before you defend the technical decision."*

- Tell me about a disagreement with a teammate / peer / senior engineer
- Tell me about a time you changed your mind
- Tell me about a time you were wrong about something technical
- Tell me about a time you pushed back on someone
- Tell me about a time you had to kill your own work
- Tell me about a time you got new information that changed a decision
- How do you handle conflict on a team?
- Tell me about a time you disagreed but committed
- **Tell me about a time you couldn't meet a commitment** *(you committed a test runner into the shared pipeline, then retracted the coverage when you couldn't keep it disruption-free — stay on the commitment you dropped, not the peer dynamic)*

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

### Rippling — Filing-Fee Automation *(not in the numbered set, but real and it scored well)*
**Opener:** *"Rippling was leaking revenue on payroll offboarding because the filing fee was still collected manually — and nobody had put a number on it."*

- Tell me about your biggest business impact
- Tell me about a time you found a problem nobody asked you to look at
- Tell me about work you did outside your assigned scope
- Tell me about a time you worked with a PM on a customer-facing decision

### Rippling — Configurable Validations Framework
**Opener:** *"Customers offboarding from a product were getting blocked by validations that didn't apply to them, and every block became a support ticket an engineer had to clear by hand."*

- Tell me about a time you reduced toil or support burden
- Tell me about designing for configurability / extensibility
- Tell me about a time you fixed a class of problem instead of an instance
- **Tell me about a time you went above and beyond for a customer** *(went past clearing tickets by hand to build the framework that removed the customer's false blocks entirely — lead with the customer being unblocked, not the toil)*

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
**Result**: Loss contained to one low-traffic geo; full recovery in 15 min; no meaningful customer impact. Migration completed zero-downtime across all geos. $156K/yr saved. Blast radius geo → island.

**Earned Secret**: Dangerous sequence is copy → delete → route. Safe sequence is copy → route → verify → **then** delete. If you delete before verifying, you've destroyed your own fallback — which is exactly what made flag-off insufficient. Also: centralize routing behind one abstraction *before* migrating, and audit from a checklist, not memory.
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
