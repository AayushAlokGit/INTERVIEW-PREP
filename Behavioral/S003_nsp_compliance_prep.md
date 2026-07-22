# S003 — NSP Security Compliance Deployment
**Strength: 4**
**Use for:** Technical judgment / "read the system before you design," ownership of a large initiative, delivering under a hard deadline, adapting to blockers, security/compliance work, large-scale infrastructure change, cross-team impact

---

## Spoken Delivery (say this version out loud)

I was a mid-level engineer on the Dynamics 365 Sales Hub Premium team when my manager assigned me to lead a company-wide cloud security initiative. Every cloud resource across 23 global Azure subscriptions had to be associated with Azure Network Security Perimeters — NSPs — to close a real data-exfiltration risk on sensitive data. And we had a hard two-month deadline with no flexibility.

The highest-leverage decision I made was refusing to design the architecture until I understood how the system actually behaved. Before committing to anything, I read the codebase myself to map the real traffic patterns for each resource type. That investigation is what shaped the whole design: Key Vaults were accessed from completely different code paths than storage accounts, which were in turn accessed by Spark jobs. Each resource type had a genuinely distinct traffic pattern.

So I considered the two obvious approaches and rejected both. One profile for everything would be too coarse to give any meaningful segregation. A unique profile per resource would be operationally unsustainable across 100-plus resources. Instead I designed profiles around traffic-pattern groupings — one profile per distinct pattern, with resources of the same type sharing a profile. That gave me least-privilege granularity where it mattered without an unmanageable number of profiles.

Before touching production, I validated the entire strategy end-to-end in a single test subscription. Then, rather than guessing at access rules upfront, I deployed in learning mode and watched the Azure dashboards after each run to see the real inbound and outbound IPs, and wrote precise rules from that live traffic. For the production rollout I coordinated daily with a senior engineer who held the subscription access — every morning I'd come in with the plan, they'd kick off the pipeline, we'd monitor it together for failures, and I'd validate after each run. After enforcement I confirmed that no existing features broke and only legitimate traffic was flowing.

Midway through, a few subscriptions failed to deploy because we didn't have enough Azure quota for the perimeters and profiles. I raised quota requests through the portal, they came back within a reasonable time, and — this is the part I'm proud of — I didn't sit idle while blocked. I finished the rollout in the other geos and started validation there, so the quota delay cost us zero calendar time against the deadline.

I also wrote up the architectural decision-making and shared it proactively in team channels, since other teams were facing the same initiative. Two engineers from other teams reached out for it.

In the end I secured over 100 cloud resources across all 23 subscriptions within the two-month deadline, with no legitimate traffic disrupted and no features broken — so the security improvement came at zero cost to reliability.

The biggest thing I took away was the value of reading actual system behavior before making architectural decisions. Without mapping real traffic first, my profile design would have been either too fine-grained or too general, and either mistake would have been costly at that scale. I now treat that upfront investigation as non-negotiable.

---

## The Story (STAR)

**Situation:**
I was a mid-level engineer on the D365 Sales Hub Premium team when my manager assigned me to lead a company-wide cloud security initiative. Every cloud resource across 23 global Azure subscriptions needed to be associated with Azure Network Security Perimeters (NSPs) to close a real data-exfiltration risk on sensitive data. We had a hard two-month deadline with no flexibility.

**Task:**
Design the NSP profile architecture and roll it out across 100+ resources and all 23 subscriptions within the deadline — applying least-privilege access rules without disrupting any legitimate traffic or breaking existing features.

**Action:**
Before committing to any architecture, I read the codebase myself to map the actual traffic patterns for each resource type. That investigation was the key move: Key Vaults were accessed from entirely different code paths than storage accounts, which were in turn accessed by Spark jobs. Each resource type had fundamentally distinct traffic patterns.

I considered two obvious approaches and rejected both. A single profile for all resources would be too coarse-grained to give meaningful segregation. A unique profile per individual resource would be operationally unsustainable at 100+ resources. Instead, I designed a profile structure based on **traffic-pattern groupings** — one profile per distinct pattern, with multiple resource instances of the same type sharing a profile. That balanced security granularity with operational simplicity.

I validated the entire deployment strategy end-to-end in a single test subscription before touching production. Rather than estimating access rules upfront, I deployed in learning mode and monitored Azure dashboards after each deployment to observe the real inbound and outbound IPs, then wrote precise rules from live traffic data. For production rollouts I coordinated daily with a senior engineer who held the necessary subscription access — each morning I asked them to kick off the deployment pipeline, we monitored it together for failures, and I validated after each run. After enforcement I verified that no existing features broke and only expected, legitimate traffic was flowing.

Midway through production, a few subscriptions failed to deploy because we didn't have sufficient Azure quota for the perimeters and profiles. I raised quota requests through the Azure portal; they were resolved within reasonable time and didn't disrupt the timeline. While those geos were blocked, I completed the rollout in other geos and started validation there — so the blocker cost us no calendar time.

I also documented the architectural decision-making and shared it proactively in team channels so other teams navigating the same initiative could benefit.

**Result:**
Secured 100+ cloud resources across all 23 Azure subscriptions within the two-month deadline. Post-deployment verification confirmed no legitimate traffic was disrupted and no existing features broke — the security improvement came at zero cost to reliability. The documentation had reach beyond my team: two engineers from other teams reached out for it.

---

## Earned Insight
> "The most important decision I made wasn't the profile design — it was reading the actual system behavior before designing anything. Without mapping real traffic patterns first, my profile structure would have been either too fine-grained or too general, and either mistake would have been costly at 100+ resources. I now treat that upfront investigation as non-negotiable before committing to any design."

**The callback (how it transferred):** On a later project I needed to pass additional CRM entity fields into an LLM's context so it could generate insights grounded in that data. Instead of building a new ingestion path, I investigated the code first and found an existing mechanism already injecting a fixed set of entity fields — I reused it to add the new fields alongside them. Reading the system before designing saved me from building something that already existed.

---

## Lead With
"I was handed a company-wide security rollout with a hard two-month deadline, and the highest-leverage thing I did was refuse to design the architecture until I'd read how the system actually behaved."
Lead with the technical-judgment move (investigate-before-design), not the compliance task itself.

---

## Technical Depth Questions (Interviewer Voice)

**NSP fundamentals:**

- "What is a Network Security Perimeter and how is it different from a VNet or NSG?"
  **Ans:** An NSP is a compliance boundary around PaaS resources — it controls which traffic is allowed in and out at the resource level. A VNet is network-level isolation for IaaS resources like VMs; NSGs are firewall rules on that network. PaaS resources like Storage Accounts, Key Vaults, and SQL Servers don't live inside a VNet by default — NSP was Microsoft's answer to applying network-level access control to those resources without requiring VNet integration.

- "What's the difference between learning mode and enforced mode in NSP? Why deploy in learning mode first?"
  **Ans:** Learning mode is an observation phase — you associate your PaaS resources with an NSP profile and traffic flows normally, but all inbound and outbound access is logged and surfaced in a dashboard. You use those logs to understand real traffic patterns and define your access rules. Enforced mode then strictly allows only the rules you've defined — anything not explicitly permitted is blocked. I deployed in learning mode first because PaaS resources often have non-obvious traffic sources; observing live traffic let me write precise rules from real data instead of guessing, and avoided blocking legitimate traffic by jumping straight to enforcement.

- "What are service tags in Azure? How are they used in traffic rules?"
  **Ans:** Service tags are named groups of IP address ranges managed by Microsoft, representing a specific Azure service or set of services. Instead of maintaining a list of IPs manually, you reference a tag — like `Storage` or `AzureMonitor` — and Azure keeps the underlying IPs up to date. In NSP access rules, service tags let you say "allow traffic from this Azure service" without hardcoding IPs that could change.

---

**Your design:**

- "How did you decide on the profile architecture — one profile per resource, one shared profile, something else?"
  **Ans:** Profiles grouped by traffic pattern. I rejected the two extremes: a single profile for everything is too coarse to give meaningful segregation, and a unique profile per resource is operationally unsustainable across 100+ resources. When I read the code I found each resource type had a distinct traffic pattern — Key Vaults, storage accounts, and Spark-accessed resources were all reached through different code paths. So I built one profile per distinct pattern and let resources of the same type share it. That gave me least-privilege granularity where it mattered without an unmanageable number of profiles.

- "How did you analyze the traffic patterns — was this manual, scripted, or tooled?"
  **Ans:** Two layers. First, I read the codebase directly to understand which code paths accessed which resource types — that's what revealed the distinct patterns and drove the profile grouping. Then, during rollout, I used learning mode: when you associate a resource with an NSP profile, all inbound and outbound traffic is logged to an Azure dashboard. I used those live logs to write the precise access rules rather than estimating them upfront.

- "How did you validate the approach before touching production?"
  **Ans:** I ran the entire deployment strategy end-to-end in a single test subscription first — profile creation, association, learning-mode observation, rule definition, and enforcement — so I'd shaken out the mechanics before any production resource was involved.

- "How did you phase the deployment across 100+ resources and 23 subscriptions?"
  **Ans:** Test subscription first to validate the whole flow, then production rolled out geo-by-geo. In each subscription I associated resources with profiles in learning mode, observed traffic, wrote rules, then moved to enforcement. I coordinated daily with a senior engineer who had the subscription access — each morning we kicked off the pipeline, watched for failures together, and I validated after each run before expanding.

- "How did you verify the rules weren't blocking legitimate traffic after enforcement?"
  **Ans:** I knew the expected access patterns per resource type from reading the code — which microservices and jobs hit them and when. After enforcement I checked the application and job-execution logs to confirm those known code paths completed without access errors. A previously-clean job suddenly failing with auth or network errors would have been the signal to investigate. Verification confirmed no legitimate traffic was disrupted and no features broke.

---

**The obstacle:**

- "Did anything go wrong during the rollout? How did you handle it?"
  **Ans:** Yes — in a few subscriptions the deployment failed because we didn't have sufficient Azure quota for the security perimeters and profiles. I raised quota-increase requests through the Azure portal, and they were resolved within a reasonable time. The important part is I didn't sit idle while blocked: I completed the rollout in the unblocked geos and started validation there, so the quota delay cost us no calendar time against the two-month deadline.

- "How did you keep a hard two-month deadline with no flexibility on track?"
  **Ans:** Front-loading the risky thinking (reading the system and validating end-to-end in a test subscription before production), parallelizing across geos so a blocker in one didn't stall the others, and a tight daily loop with the senior engineer who held access — kick off, monitor, validate, expand. Nothing waited on a weekly sync.

---

**Design critique:**

- "If you were designing this from scratch today, what would you do differently?"
  **Ans:** I'd define the access rules as code from the start. Writing rules from live traffic worked, but doing it across 100+ resources by hand is error-prone and hard to audit. Rules-as-code would make the whole rollout repeatable and reviewable. I'd also start learning-mode observation across all subscriptions as early as possible in parallel, to get the complete traffic picture sooner.

- "How would you detect if the rules were allowing traffic they shouldn't, or blocking traffic they shouldn't?"
  **Ans:** Blocking legitimate traffic surfaces quickly — application errors, failed jobs, access-denied logs. Allowing traffic it shouldn't is harder; you'd want anomaly detection on the access logs to flag traffic from unexpected sources. I'd treat that as ongoing monitoring rather than a one-time verification.

---

## Behavioral Follow-Up Questions (Interviewer Voice)

- "Why did you start by reading the code instead of just designing the profiles?"
  **Ans:** Because the profile architecture depends entirely on how traffic actually flows, and that wasn't obvious from the outside. If I'd guessed, I'd have landed on one of the two bad extremes — one profile for everything, or one per resource. Reading the code showed me the resource types had genuinely distinct access patterns, which is what made the traffic-pattern-grouping design the right call. The design fell out of the investigation.

- "You were mid-level leading a company-wide initiative — how did you handle the scope and the senior engineer you depended on?"
  **Ans:** I owned the design and the validation; I depended on the senior engineer for subscription access. So I made that dependency low-friction — a predictable daily loop where I came in with the plan for the day, they kicked off the pipeline, and we monitored together. I did the validation and documentation so their involvement stayed narrow. Treating their access as a scarce resource and organizing around it kept a big rollout moving.

- "How did the documentation end up helping other teams?"
  **Ans:** Other teams were facing the same NSP initiative, so I wrote up the decision-making — why traffic-pattern grouping, how learning mode informs the rules, how to sequence the rollout — and shared it proactively in team channels rather than keeping it as personal notes. Two engineers from other teams reached out for it. It cost me little to write and saved them the discovery work I'd already done.

- "What did you take away from this that you've used since?"
  **Ans:** Read the actual system behavior before making an architectural decision. It directly helped me later on an LLM feature — I needed to pass extra CRM entity fields into the model's context, and instead of building a new ingestion path I read the code first and found an existing mechanism already injecting fixed entity fields. I reused it to add the new ones. Same instinct: investigate before you build, because the answer is often already in the system.

---

## Watch Out For

- "Wasn't reading the whole codebase overkill for a config rollout?" → It wasn't a config rollout — the profile architecture *was* the deliverable, and it depended on traffic patterns that weren't visible without reading the code. The investigation is what prevented a costly wrong design at 100+ resources.
- "The quota failure sounds like poor planning." → Quota ceilings on a brand-new resource type across 23 subscriptions aren't fully knowable upfront; the signal of ownership is that I unblocked it fast and parallelized so it cost zero calendar time.
- Don't undersell the design reasoning. The interesting part isn't "I applied NSPs" — it's that I reasoned about a spectrum (too coarse ↔ too granular) and deliberately picked the operationally sustainable middle, driven by evidence from the code.
- Keep the senior engineer framed accurately: a collaborator who held subscription access and whom I coordinated with daily — not a blocker and not a decision-maker over the design.
