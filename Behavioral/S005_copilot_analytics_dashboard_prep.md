# S005 — Copilot Usage Analytics Dashboard
**Strength: 4**
**Use for:** Learning unfamiliar tech fast under deadline, knowing when to ask for help, ambiguity / requirements gathering, cross-team coordination + dependency management, stakeholder communication, self-awareness / growth (a habit I fixed), sole-owner end-to-end delivery, performance investigation, GenAI-adjacent work

---

## Spoken Delivery (say this version out loud)

On the Dynamics 365 Sales team, I was the sole engineer responsible for designing and delivering an analytics dashboard that gave internal leadership their first visibility into Copilot skill usage trends. The catch was the setup: a 1.5-month deadline, a 1TB daily telemetry stream, and zero prior experience with any of the systems involved.

I started by going straight to leadership to lock down requirements — specifically which Copilot skills to track and which metrics actually mattered. Getting that clarity up front meant I could move fast without second-guessing scope later. Then I hit the hard part: I had to learn an internal big-data job platform and an internal SQL-like language called SCOPE completely from scratch. I read the internal docs, but I also recognized early that struggling alone was slower than finding the right person — so I tracked down a senior engineer who knew both tools and used their guidance to cut through the ambiguity and get to the right resources faster.

Once I had footing, I built a pipeline that processed the daily telemetry streams and wrote the results to Azure Blob Storage, which fed the Power BI reports embedded in the dashboard UI. Integrating with the Power BI infrastructure team brought its own challenges — I onboarded onto their API documentation, figured out how to trigger report refreshes and assign the right permissions, and then ran load experiments with roughly 200GB of dummy data — a rough capacity estimate based on our largest customer — to test the initial-refresh scenario, when the report loads its data for the very first time.

Those experiments surfaced a refresh time of around three hours, which was unacceptable for a daily-updating dashboard. I brought that finding to the Power BI team with concrete data, which gave them exactly what they needed to investigate and fix it. While they worked on it, I kept moving — running end-to-end tests on lower data volumes to validate the full pipeline so I wasn't blocked waiting on them.

Throughout, I posted updates in a shared team channel where leadership was also present. When the refresh-time issue came up, I raised it there too — the load test, the finding, the optimization effort on the Power BI side, and the honest risk that it could delay the release. Fortunately the Power BI team turned it around quickly.

One thing I let drag too long was cross-timezone coordination. I was reluctant to schedule late-night meetings, and that reluctance was quietly slowing things down. Once I recognized it, I stopped accommodating my own schedule and started booking the earliest slot that worked for the Power BI team.

The dashboard shipped within the 1.5-month deadline. The Power BI team's optimizations brought refresh time down from three hours to around one, which made automated daily updates practical, and leadership had day-over-day and month-over-month visibility into Copilot skill usage for the first time. The project was later deprioritized as the org shifted toward agentic features, but the working deliverable was completed and handed off successfully.

The clearest thing I took away was about timezone friction. My reluctance to schedule late-night meetings introduced delays that were completely avoidable. I learned to treat cross-timezone coordination as a logistics problem to solve immediately, not a personal inconvenience to minimize — and I'd act on that much earlier if I did it again.

---

## The Story (STAR)

**Situation:**
On the Dynamics 365 Sales team, I was the sole engineer assigned to design and deliver an analytics dashboard giving internal leadership their first visibility into Copilot skill usage trends. Three things made it hard at once: a 1.5-month deadline, a 1TB daily telemetry stream, and zero prior experience with any of the systems involved.

**Task:**
Own the whole thing end-to-end — nail down requirements, learn the data platform, build the telemetry pipeline, integrate with Power BI, and ship a daily-updating dashboard within the deadline.

**Action:**
I went to leadership first to lock down scope — which Copilot skills and which metrics — so I wouldn't churn on scope later. Then I taught myself an internal big-data job platform and its SQL-like language, SCOPE, from scratch — but instead of grinding alone, I found a senior engineer who knew both tools and used their guidance to move faster. I built a pipeline that processed the daily telemetry and wrote results to Azure Blob Storage, feeding Power BI reports embedded in the dashboard UI. Integrating with the Power BI infra team, I learned their API, worked out report-refresh triggering and permissions, and load-tested the initial-refresh scenario with ~200GB of dummy data (sized to our largest customer). That revealed a ~3-hour refresh — unacceptable — so I brought concrete data to the Power BI team to drive the fix, and kept the pipeline moving on lower volumes while they optimized. I communicated progress and risks (including the refresh issue) in a shared channel with leadership present. Mid-project I caught myself under-scheduling cross-timezone meetings to protect my own hours, recognized it was slowing us down, and switched to booking the earliest slot that worked for the Power BI team.

**Result:**
Shipped within the 1.5-month deadline. Power BI optimizations cut refresh time from ~3 hours to ~1 hour, making automated daily updates practical. Leadership got first-ever day-over-day and month-over-month visibility into Copilot skill usage. The project was later deprioritized in favor of agentic features, but the working deliverable was completed and handed off successfully.

---

## Earned Insight
> "Cross-timezone coordination is a logistics problem to solve immediately, not a personal inconvenience to minimize. My reluctance to book late-night meetings introduced delays that were entirely avoidable — the friction wasn't the timezones, it was me optimizing for my own comfort over the project's timeline. Now I book the earliest slot that works for the other team and treat that as the default."

A second, complementary insight: **struggling alone is a false economy.** With two systems to learn from scratch under a tight deadline, the fastest path wasn't reading every doc — it was finding the one person who already knew the tools and using their guidance to get to the right resources. Knowing *when* to ask is a skill, not a crutch.

---

## Technical Depth Questions (Interviewer Voice)

**The pipeline & platform:**

- "Walk me through the data flow end-to-end — from 1TB of daily telemetry to what leadership saw."
  **Ans:** Raw Copilot usage telemetry landed as a daily stream — about 1TB/day. I wrote jobs on an internal big-data platform, using a SQL-like language called SCOPE, to process that stream: filter to the skills leadership cared about, aggregate into the metrics we'd agreed on, and roll them up by day and month. The processed results were written to Azure Blob Storage, and Power BI reports read from Blob to render the visuals. Those reports were embedded directly in the dashboard UI leadership used.

- "What is SCOPE and how did you approach learning it with no prior experience?"
  **Ans:** SCOPE is an internal SQL-like language for large-scale data processing on Microsoft's big-data platform — declarative, set-based, similar in feel to SQL but built for massive distributed jobs. I learned it from the internal documentation, but the accelerator was pairing that with a senior engineer who already used it — they pointed me to the right patterns and resources so I wasn't discovering everything by trial and error. That combination got me productive far faster than docs alone.

- "Why write to Azure Blob Storage as the intermediate layer instead of having Power BI query the source directly?"
  **Ans:** Blob decoupled the heavy processing from the reporting layer. The big-data jobs did the expensive aggregation once per day and wrote a compact, report-ready result set to Blob; Power BI then just read that pre-aggregated output. That kept Power BI from having to touch the full 1TB stream and gave a clean, stable hand-off boundary between my pipeline and the Power BI side.

- "How often did the pipeline run, and what triggered the Power BI report to refresh?"
  **Ans:** It ran daily, aligned to the daily telemetry drop. Once the processed results were written to Blob, the Power BI report needed to refresh to pick up the new data. I used the Power BI team's API to trigger those refreshes and to set the right permissions so the refresh could read from our Blob storage. Getting that refresh reliable and automated is what made it a genuinely daily dashboard rather than a manual one.

---

**The performance investigation:**

- "How did you find the refresh-time problem, and why 200GB of dummy data specifically?"
  **Ans:** I load-tested the initial-refresh scenario — the first time a report loads its full dataset, which is the worst case. I generated roughly 200GB of dummy usage data as a rough capacity estimate for our largest customer, so I was testing near the top of the realistic range rather than a happy-path small sample. That test showed the initial refresh taking around three hours, which wouldn't work for a dashboard meant to update daily.

- "Why was the initial-refresh scenario the one you chose to stress-test?"
  **Ans:** Because it's the heaviest case — loading the entire dataset cold, with no prior state to build on. If the first full load was fine, incremental daily updates would be lighter. Testing the worst case first meant I'd surface the scaling problem early instead of discovering it after launch when a big customer's data hit the report.

- "You said you 'brought the finding to the Power BI team' — why not fix it yourself?"
  **Ans:** The bottleneck was inside the Power BI refresh infrastructure, which that team owns — not in my pipeline. The most useful thing I could do was give them a precise, reproducible finding: here's the scenario, here's the data volume, here's the three-hour number. Concrete data is what let them investigate and fix it efficiently. Meanwhile I de-risked my own side by running end-to-end tests on lower volumes so the rest of the pipeline was validated and ready.

- "What did they change to get it from three hours to one, and were you involved?"
  **Ans:** The optimizations were on the Power BI refresh side — their domain, so I won't overstate the internals. My role was to surface the problem with hard numbers, keep them unblocked with reproducible test data, and validate the end-to-end result once it improved. The outcome was refresh dropping from ~3 hours to ~1 hour, which made a reliable daily automated refresh practical.

---

**Design critique:**

- "If you did this again, what would you change technically?"
  **Ans:** I'd load-test the refresh scenario against the Power BI infrastructure much earlier — before building out the full pipeline — since that turned out to be the riskiest integration point. Finding the three-hour refresh sooner would have given the Power BI team more runway. I'd also formalize the cross-timezone coordination from day one, because that dependency was on the critical path and I let it run slower than it needed to.

- "How would you have handled it if the Power BI team couldn't optimize the refresh in time?"
  **Ans:** I'd have looked at reducing what the refresh had to do — pre-aggregating more aggressively in the pipeline so Power BI loaded a smaller result set, or moving to incremental refresh so only new data loaded each day instead of a full reload. The three-hour number was on a cold full load; a daily incremental would touch far less data. So there was a fallback path on my side even if theirs hadn't landed in time.

---

## Behavioral Follow-Up Questions (Interviewer Voice)

- "You had zero experience with these systems and 1.5 months. When did you decide to ask for help instead of pushing through the docs?"
  **Ans:** Early — and deliberately. I read the internal docs first to get oriented, but I noticed I was spending a lot of time just figuring out *what* to read. That's the signal that a knowledgeable person will be faster than more searching. So I found a senior engineer who knew both the platform and SCOPE and used their guidance to get to the right resources. It wasn't outsourcing the work — it was using their map instead of drawing my own from scratch.

- "Tell me about the timezone thing — that's an unusual thing to volunteer."
  **Ans:** The Power BI team was in a different timezone, and syncing with them often meant a late-night meeting for me. I was quietly avoiding scheduling those, telling myself we'd catch up async — but async was slow, and that dependency was on the critical path. Once I noticed my own reluctance was the bottleneck, I changed it: I started booking the earliest slot that worked for them, even when it was inconvenient for me. I volunteer it because it's the clearest thing I actually changed about how I work.

- "How did you handle requirements when leadership hadn't built anything like this before?"
  **Ans:** I went to them directly and made them make the scoping decisions up front — which Copilot skills to track, which metrics mattered. I didn't want to guess and rebuild. Locking that down early is what let me move fast on the hard technical parts without worrying the scope would shift under me.

- "How did you keep leadership and stakeholders in the loop?"
  **Ans:** I posted regular updates in a shared team channel where leadership was present, so status was visible by default rather than on request. When the refresh-time risk came up, I raised it in that same channel — the load test, the finding, the Power BI team's optimization effort, and an honest flag that it could risk the release date. I'd rather surface a schedule risk early with a plan attached than have it be a surprise later.

- "The project got deprioritized. How do you feel about having built something that got shelved?"
  **Ans:** The deliverable was completed, working, and handed off within the deadline — the deprioritization was an org-level strategy shift toward agentic features, not a reflection of the work. I'm comfortable with that: I delivered what was asked, on time, and the skills I built — a new data platform, SCOPE, Power BI integration, driving a cross-team performance fix — carried straight into later work. Business priorities move; execution is what I control.

- "What did you learn about working across teams from this?"
  **Ans:** Two things. Give a dependent team a precise, reproducible problem, not a vague complaint — concrete data is what let the Power BI team fix the refresh fast. And treat coordination logistics, like timezones, as something to solve immediately rather than tolerate. The technical work was rarely the thing slowing us down; the coordination was.

---

## Watch Out For

- "Did you optimize the refresh yourself?" → Be honest: the refresh optimization was the Power BI team's work in infrastructure they own. Own your real contribution — you *found* it with a load test, quantified it, drove it with data, and stayed unblocked meanwhile. Don't claim their fix.
- "Sole engineer — so you did everything?" → Yes on the pipeline, requirements, and integration; but you depended on a senior engineer to ramp on the tools and on the Power BI team for the refresh. Frame using them as good judgment, not a gap.
- Don't bury the timezone lesson as a throwaway — it's a genuine self-aware "habit I changed" beat, which is exactly what "tell me about a weakness / something you'd do differently" questions want. Lead the reflection with it.
- "The project was deprioritized" — say it plainly and early if relevant; don't let it sound like the work failed. Deliverable completed and handed off; priority shift was org-level.
- Don't overstate the GenAI depth — this is *analytics about* Copilot usage, not building the Copilot skills themselves. S004 (LLM agent E2E testing) is your hands-on-LLM story; your MCP/RAG documentation-assistant side project is your deepest GenAI talking point if asked for model-level work.
