# S008 — BCDR Drill: The Buffer That Wasn't There
**Strength: 3 — short story, cap relief only**
**Use for:** preventing a problem before it happened, being proactive, not following the documented process, escalating to another team, on-call experience, disaster recovery / resilience testing

> **What this story is for.** It is **not** a lead story and you should never open a round with it. Its job is **load-balancing**: when S001 or S003 has already spent its two uses and another probe lands on proactivity or standards, this exists so you have a real answer instead of telling the config migration a fourth time. 90 seconds, self-contained, nothing to defend that you can't defend.

---

## Spoken Delivery (say this version out loud)

At Microsoft, teams owning microservices deployed across paired availability zones had to run a periodic BCDR drill — business continuity and disaster recovery. The platform team injects failure into the nodes of one availability zone, and the owning team has to verify that traffic fails over to the redundant zone with no loss of availability. I was on-call for my team, so running our service's drill was mine. It was my first one.

The way you verify availability is by hitting the service's regional endpoints and checking you get valid responses. But those endpoints were behind strict auth — you couldn't just curl them. They could only be called from a properly authenticated environment, and that environment had to be provisioned in the same region where the fault was going to be injected. So the drill had a hard dependency: no test environment in that region, no drill.

We had documentation for setting that environment up, and it recommended doing it the day before the deadline. I didn't take that. It was my first drill and I wanted margin, so I ran the setup a week out. Provisioning failed. I checked I hadn't misread the doc, then raised an incident on the BCDR team with the provisioning failure logs attached — which is why they picked it up immediately instead of bouncing it back as a configuration problem on my end. They diagnosed it as a capacity shortage for test environments in that region. Not something specific to my team — a region-level limit. It took three days to resolve, and my buffer absorbed it.

The drill then ran on schedule. The failover was seamless — traffic moved to the second availability zone, no availability drop, nothing broken in our service.

The part I actually think about is the arithmetic. The documentation recommended a one-day buffer, and the problem sitting in it took three days to clear. So the documented process had a buffer one-third the size of the failure it needed to absorb — and because it was a regional capacity limit, anyone provisioning a test environment in that region would have hit the same wall. Following the runbook exactly would have made the drill impossible to run.

And the honest gap is that I didn't fix that. I concluded the doc was fine, because the steps in it were correct. But the steps weren't the problem — the recommended timing was, and I'd just proven it. I left the next on-call to find that out for themselves.

---

## The Story (STAR)

**Situation:**
At Microsoft, teams owning microservices deployed across paired availability zones in a region were required to run a periodic BCDR drill: the platform team injects failure into the nodes of one AZ, and the owning team verifies traffic reroutes to the redundant AZ with no availability loss. I was the on-call for my team and responsible for carrying out the drill for our microservice. It was my first.

**Task:**
Prove our microservice remained available through the fault-injection window, against a hard drill deadline.

**Action:**
Verifying availability meant calling the service's regional endpoints and checking for valid responses — a valid response meant up, no response meant down. The endpoints were behind strict auth and couldn't be hit manually with curl; they could only be called from a properly authenticated environment, and that environment had to be provisioned **in the same region as the fault injection**. On-calls had documentation for that setup, and it recommended doing it **the day before** the deadline.

Being cautious on a first drill, I ran the setup **a week out** instead. Provisioning failed in the target region. I confirmed I hadn't misfollowed the documentation, then raised an incident on the BCDR team **with the provisioning failure logs attached** — which is why it was accepted on first contact rather than returned as a configuration issue on my side. The BCDR team diagnosed it as a **capacity shortage for test environments in that region** — a region-level limit, not team-specific. It took **three days** to resolve. My buffer covered it.

**Result:**
The drill was carried out on the deadline. Failover to the second availability zone was seamless — no availability drop, no features broken. The substantive result is the counterfactual: the documented process recommended a **one-day** buffer for a failure that took **three days** to clear, and because the constraint was regional capacity, any team provisioning a test environment there would have hit it. Following the documentation exactly would have made the drill impossible to run.

---

## Earned Insight
> "A documented prep timeline is only safe if its buffer is bigger than the time it takes to recover from the most likely failure inside it. Here the buffer was one-third the size. And the real dependency wasn't the drill — it was **provisioning capacity in someone else's system**. You can't discover a capacity limit at the moment you need the capacity."

Say this out loud. It reframes the story from "I like starting early" — a personality trait — into a judgment about where the risk actually sat.

---

## Follow-Up Questions (Interviewer Voice)

**"Why did you start a week early when the documentation said a day was enough?"**
Honestly, caution — it was my first drill and I didn't have the experience to know which steps were risky, so I gave myself margin on all of them. I want to be straight that I didn't predict this specific failure. What I'd defend is the principle: the setup had a hard dependency on provisioning capacity in another team's system, and any dependency you don't control is worth testing before the day you need it.

**"What did you do between the provisioning failing and filing the incident?"**
Confirmed the failure was reproducible and that I hadn't misfollowed the documentation — the setup steps were prescriptive enough that a misstep was the first thing to rule out. Once I was confident it wasn't on my side, I collected the provisioning failure logs and raised the incident with them attached. That's why it was accepted immediately instead of coming back to me as a config problem.

**"Did you diagnose the capacity issue yourself?"**
No — the BCDR team did. I want to be accurate about that. What was mine was reproducing it, ruling out my own error, and escalating with enough evidence that they could act on it without a round-trip.

**"What would have happened if you'd followed the documentation and set up the day before?"**
The environment wouldn't have provisioned, and the fix took three days — so the drill couldn't have been carried out. There'd have been no way to verify availability during the fault injection at all.

**"Did other teams hit this?"**
I don't know whether anyone else did, and I don't want to claim I rescued anybody. What I can tell you is the mechanism: it was a capacity limit on test environments **in that region**, not something specific to our team's configuration. Any team that needed to provision one there would have run into the same wall.

**"Did the drill itself find anything?"** / **"What was the result of the fault injection?"**
The failover worked. Traffic moved to the redundant availability zone, there was no availability drop, and nothing in our service broke. Nothing to fix afterwards — which was the point of running it.

**"So you updated the documentation afterwards?"** ⚠️ *Don't wait to be asked — volunteer this.*
No, and that's the thing I'd change. I concluded the doc was fine because the steps in it were correct. But the steps weren't the problem — the recommended timing was, and I'd just demonstrated that a one-day buffer couldn't absorb the most likely failure in it. The whole value of what I found was in the timing recommendation, and I left the next on-call to rediscover it themselves. The fix was one line in a runbook.

**"How would you run a BCDR drill differently now?"**
I'd work backwards from the dependencies rather than from the deadline. The drill day is fixed, but everything that has to be true before it — a provisioned environment, auth, access — sits in systems other teams own, and those are where the time goes. I'd validate the external dependencies first and leave the parts I control until last, because those are the ones I can compress if I'm short on time.

---

## Watch Out For

- **Lead with the arithmetic, not with being early.** Open on *"the runbook said set up one day ahead; the problem I found took three days to fix."* Opening on "I like to start early" makes it a personality trait rather than judgment, and there's nothing to score in a personality trait.
- **Don't claim you saved other teams.** You don't know that anyone else hit it. Give the mechanism — regional capacity limit — and stop there.
- **Don't overstate the diagnosis.** The BCDR team found the root cause. Yours: reproduced it, ruled out your own error, escalated with logs attached. The logs detail is the defensible one — it's why the incident was accepted on first contact.
- **Volunteer the documentation gap before you're asked.** It's the strongest beat in the story and the only genuine self-criticism in it. Held back, it becomes a hole an interviewer finds. Offered, it reads as Ownership.
- **There is no failure or conflict beat here.** Failover worked, drill passed, nobody pushed back on you. Don't manufacture drama — the entire point is that nothing happened.
- **Not an Ownership-outside-your-scope story.** You were the on-call; the drill was assigned. The judgment was in the timing, not in taking the work on. For "outside my scope," this is not the story.
- **Keep it to 90 seconds.** It's a 3. Told at four minutes it competes for airtime with stories worth twice as much.

---

## Unknowns — answer honestly, don't reconstruct

- Whether any other team actually hit the capacity limit. **You don't know.**
- Why the region was short on capacity — whether it was a quota, a hardware constraint, or contention from other teams drilling at the same time. **The BCDR team didn't tell you and you didn't ask.**
- Whether the capacity limit recurred for later drills, or whether the documentation was ever updated by someone else. **You don't know.**

"I don't know — that sat with the BCDR team" costs nothing here. Guessing at another team's infrastructure costs the question.
