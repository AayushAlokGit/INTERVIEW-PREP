# Behavioral Prep — Aayush Alok
Last updated: 2026-04-19

---

## Story Quick Reference

| ID | Title | Strength | Use When |
|----|-------|----------|----------|
| S001 | Zero-Downtime Config Migration | 4.5 | When in doubt — most versatile story |
| S003 | NSP Security Compliance | 4 | Technical decision challenged, ambiguity |
| S004 | Personal RAG Assistant | 4 | Initiative, GenAI, personal projects |
| S002 | Payroll Fee Automation | 3 | Business impact (backup only) |

---

## Coverage Map

| Category | Story / Script |
|---|---|
| Technical depth / decision-making | S001, S003 |
| Production incident + recovery | S001 |
| Failure / learning | S001 |
| Admitting a mistake | S001 |
| Receiving feedback | S001 (reframed) |
| Communication under pressure | S001 |
| System reliability / zero-downtime design | S001 |
| Technical decision challenged by senior | S003 |
| Disagreement with senior engineer | S003 |
| Security / compliance tradeoff | S003 |
| Ambiguity navigation | S003 |
| Cross-team coordination | S003 |
| Large-scale infrastructure change | S003 |
| Initiative / self-directed work | S004 |
| GenAI / LLM experience | S004 |
| Personal projects | S004 |
| Pragmatic decision-making | S004 |
| Business impact | S002 |
| Process improvement | S002 |
| Conflict / difficult colleague | Script |
| Giving feedback | Script |
| Prioritization under pressure | Script (anchored to S001) |
| "Why did you leave Microsoft?" | Script |

---

## Stories

### S001 — Zero-Downtime Config Migration *(Strength 4.5 — Lead story)*

**Situation**: Our microservices at Microsoft used Azure Table Storage as a geo-scoped config store — high COGS and a large blast radius. If geo-level storage went down, every island in that geo lost the configs scheduling critical data processing jobs, breaking analytics across the org.

**Task**: Design and execute a zero-downtime migration to PPMS — a Cosmos DB-based tool at island scope — cutting costs by $156K/year and shrinking fault isolation boundaries.

**Action**: I built a background migration service that copied data from Table Storage to PPMS, inserted a completion flag, then cleaned up the old Table Storage data. At every write path, I added a flag check to redirect writes to PPMS post-migration. I controlled rollout per-geo via feature flags, starting with low-traffic first-release geos. During that rollout, I discovered a critical mistake: I'd missed the flag check in one write path, and the cleanup had already run — config data was actually missing from Table Storage in that geo. I admitted the fault directly to my manager without waiting or minimizing it. He advised: pause all migration, and recover the missing data by copying equivalent config from a neighboring geo (configs are similar across geos). I executed both, verified recovery within 15 minutes, then audited every write path in the codebase before resuming rollout.

**Result**: Full recovery in 15 minutes, no service impact. Migration completed across all geos with zero downtime. $156K/year cost reduction. Blast radius shrunk from geo-level to island-level.

**Earned Secret**: The dangerous migration sequence is copy → delete source → route writes. If write routing has any bug, you've already deleted your fallback. The safe sequence: copy → route → verify writes → then delete.

**Watch out for**: "Why didn't tests catch this?" → Test environment didn't have enough write traffic. "What did you do after fixing it?" → Audited every write path before resuming — didn't assume it was the only missed check.

---

### S003 — NSP Security Compliance Deployment *(Strength 4)*

**Situation**: Microsoft launched a compliance initiative requiring Network Security Perimeters on all PaaS resources to prevent data exfiltration. My team had 100+ resources across 10+ subscriptions globally.

**Task**: Design the NSP profile architecture and implement traffic rules with minimal scope — least-privilege network access.

**Action**: Analyzed all active resources, designed the profile architecture, deployed in monitoring mode to observe real traffic patterns. Discovered PaaS resources were receiving traffic from Microsoft's control plane — couldn't pinpoint precise service tags for it. Initial call: use the broader "MicrosoftPrivateIp" tag. A senior engineer challenged this — asked me to reach out to the NSP team to see if a narrower scope was possible. I did. Even they couldn't identify more precise service tags for control plane traffic. Brought that finding back to the senior engineer; we reviewed it together and agreed to proceed with the broader tag — documented as a known limitation with the NSP team's confirmation that narrowing wasn't currently possible. Deployed across all resources in controlled phases.

**Result**: Full compliance achieved across 100+ resources, 10+ subscriptions. Broader tag decision backed by due diligence, not assumption.

**Earned Secret**: The difference between a lazy broad allowlist and a defensible one is the paper trail of due diligence. When you can't narrow scope, document that you tried, who you consulted, and why narrowing wasn't possible.

**Lead with**: "A senior engineer challenged my security decision mid-implementation — here's what I did." (Don't lead with the compliance task.)

**Watch out for**: "Why didn't you escalate further?" → Both the senior engineer and NSP team agreed there was no further authoritative source. Documented the decision and flagged for revisiting.

---

### S004 — Personal RAG Documentation Assistant *(Strength 4)*

**Situation**: As an engineer who documents everything, I'd accumulated 375+ markdown files over time. Finding relevant information meant manually searching through dozens of files — breaking flow and costing ~2-3 hours per week.

**Task**: Build a natural-language search system over my personal documentation that integrated into my existing GitHub Copilot workflow.

**Action**: Built a 3-phase ingestion pipeline (discovery → processing → indexing) that chunked .md files with overlapping fixed-size chunks and converted them to embeddings via Azure AI Foundry. Built an MCP server to query the vector DB and surface results inside GitHub Copilot. Initially used Azure AI Search — burned through credits quickly. Migrated to locally hosted ChromaDB, same interface, zero cost. Set 95% relevance threshold as a starting heuristic — high precision over recall, since documentation volume fit in context window.

**Result**: Sub-100ms query response times. Saved ~2-3 hours per week previously spent manually searching. 375+ .md files indexed.

**Earned Secret**: A high relevance threshold that sometimes returns nothing beats a low threshold that floods context with noise. For RAG, knowing when not to retrieve is as important as retrieval quality.

**Watch out for**: "Built with GitHub Copilot" — own the architecture; Copilot was an accelerant, not the engineer. "Why not just use grep?" → Semantic similarity catches related concepts even when exact terms don't match.

---

### S002 — Payroll Fee Automation *(Strength 3 — Backup only)*

**Situation**: Rippling was systematically missing the $1,000 offboarding filing fee from customers leaving the payroll product — a manual process with no enforcement, causing silent revenue leakage.

**Task**: Automate fee collection within the existing offboarding workflow.

**Action**: Integrated a blocking step into the offboarding workflow that fired a charge event to the Payment team's infrastructure at the point of customer departure, automatically collecting the required fee.

**Result**: Recovered $45K in previously leaked revenue and eliminated the manual collection gap entirely.

**Earned Secret**: Silent revenue leakage lives at process handoffs — where one team's responsibility ends and another's begins.

**Note**: Problem was surfaced by mentor, not self-identified. Error handling was Payment team's responsibility. Do not oversell ownership or technical complexity.

---

## Gap-Handling Scripts

### Conflict / Difficult Colleague
> "I haven't been in a situation where a disagreement escalated into real conflict — most technical disagreements I've been part of got resolved through discussion pretty quickly. What I can tell you is how I approach those moments: I try to understand the tradeoffs the other person is optimizing for before I push my position, because usually disagreements are about different constraints, not different competence. The clearest example I have of navigating ambiguity with a stakeholder is my config migration incident — I admitted fault directly to my manager and we worked through the recovery together."

If pushed for peer conflict specifically: *"At my stage I haven't had a major interpersonal conflict — I've mostly been heads-down on technical execution. That's a gap I'm aware of."*

---

### Giving Feedback
> "I haven't been in a formal position to give feedback yet — most of my feedback has been peer code review comments. What I can tell you is my principle: specific and actionable, not vague. I'd rather say 'this function will fail if the input is null — here's the fix' than 'the code needs improvement.'"

---

### Receiving Feedback
Deploy **S001** — when Aayush admitted fault to his manager and the manager gave direct corrective guidance on recovery. Accepted without defensiveness, acted immediately. That IS the receiving feedback story.

---

### Prioritization Under Competing Demands
> "The clearest example I have is during my config migration incident at Microsoft. When I discovered the data was missing mid-rollout, I suddenly had four things that all felt urgent: fix the code, resume migration, recover the affected geo, and communicate to my manager. The sequence wasn't hard once I asked 'what's the blast radius of waiting on each?' — stop new damage first (pause migration), recover existing damage second (cross-geo data copy), fix the root cause third (the PR), validate nothing else was broken fourth (write path audit). I've carried that framework forward: when everything feels urgent, sequence by blast radius, not by effort."

If pushed for project-level prioritization: *"At my stage I've mostly been executing on a defined backlog. My judgment has been about sequencing within a task, not between competing projects — that's something I'm looking forward to developing at the next level."*

---

### "Why did you leave Microsoft?"
> "I'm a US citizen and my family is based here. After building 3.5 years of experience across Rippling and Microsoft, I felt I had enough foundation to make the move to the US at the right time — both personally and professionally. It was a deliberate decision, not a reactive one."

Follow-up "Why now?": *"My mother has been in the US for a while. I wanted to wait until I had enough experience to be competitive in the US market — mid-level with Microsoft and Rippling on the resume felt like the right moment."*
