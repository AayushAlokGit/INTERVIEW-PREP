# System Design Round Transcript
**Date:** 2026-08-09
**Start Time:** 17:59:11 · **End Time:** 19:26:47 · **Duration:** 88 min
**Problem:** GitHub Actions — CI/CD Workflow Execution Platform
**Difficulty:** Hard
**Dominant pattern:** long-running tasks (with multi-step processes)
**Performance Rating:** 1/5  <!-- machine-read on future rounds; <=3 = eligible for re-ask, >=4 retired -->

**Would it have fit a real 45-min round?** No — cut off at Requirements. At the 45-minute mark he was mid-way through core entities; HLD and deep dive were never reached.

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual (cumulative) | Delta | On pace? |
|---|---|---|---|---|
| Requirements | 8 min | 38 min | +30 | No |
| Core entities | 12 min | 63 min | +51 | No |
| API design | 17 min | 76 min (incomplete) | +59 | No |
| High-level design | 27 min | never reached | — | No |
| Deep dive | 40 min | never reached | — | No |
| Wrap-up | 45 min | 88 min (candidate quit) | +43 | No |
| **Total** | 45 min | 88 min | +43 | No |

---

## Conversation Log

**Interviewer:** Design GitHub Actions — the CI/CD platform built into GitHub. A user pushes a commit (or opens a PR) to a repository. That repo contains one or more workflow definition files (YAML) that say: on this event, run these jobs; each job runs on a fresh machine of a given type and executes a sequence of steps; jobs can depend on other jobs. The platform must pick up that event, provision compute, run the jobs, stream logs back to the user's browser live, store artifacts, and report a pass/fail status back onto the commit and PR. Design the system end to end. Difficulty: Hard. Reference timeline stated (measured, not enforced). Canvas path given. Start with requirements.

**Aayush:** Our system will have to parse YAML files so will there be APIs exposed which the system can trigger to fetch the YAML files of the repos?

**Interviewer:** Yes — assume GitHub itself (the git storage / repo service) is an existing internal service you can call. You can read the workflow YAML from the repo at a given commit SHA, and you can subscribe to or receive repository events (push, pull_request, etc.). Treat that as a dependency you consume, not something you build. Parsing the YAML into an executable plan is in scope. Now give me your FRs and NFRs with numbers.

**Aayush:** FRs ->
1. Parse workflow YAML files to create event watchers.
2. When event trigger detected schedule proper workflows to execute on a fresh machine of type specified in jobs YAML file.
3. Resolve order of execution of jobs, for jobs with dependencies on other jobs, from the YAML files.
4. Provision compute for the jobs depending on YAML files.
5. Execute the jobs in compute in proper sequence.
6. Stream logs of job execution back to users browser live.
7. Store artifacts generated during job execution.
8. Report pass/fail back onto commit and PR which triggered event.

**Interviewer:** Solid FR list. Two things: (1) What are you explicitly NOT designing? (2) NFRs with numbers — how many repos, workflow runs/day, peak-vs-average, typical job duration, log volume, and latency between commit pushed and job start?

**Aayush:** NFRs ->
1. Low latency job scheduling post event (<1s)
2. No event should be dropped.
3. Highly available for event receiving.
4. Atleast once guarantee for job execution.
5. Durably store artifacts for 7 days.
6. Assume 100M DAU, and each user opens 1 PR a day, 10 commits/day -> 1k PR events/s, 10K commits/s. At peak 5x. At peak 55k jobs scheduled/s. Assuming each job generates 1Mb of artifacts -> 55GB/s of artifact data generated.

**Interviewer:** (1) You skipped the scoping question — what are you NOT designing? (2) You went from 11k events/s to 55k jobs scheduled/s — is one event one job? (3) How long does a job run, and what does that imply about fleet size? (4) Log volume and freshness target?

**Aayush:**
2. Yes one job per event.
3. A job can run for a few mins to a few hours as well.
4. Log latency (<500ms). Assuming each job generates 1kb logs, 55MB/s at peak.

**Interviewer:** Pick an average — say 5 minutes. At 55k jobs starting per second, each holding a fresh machine for 5 minutes: how many machines are running concurrently at peak? Do the arithmetic and tell me whether you believe that number. And I still don't have your out-of-scope list.

**Aayush:** Out of scope -> analytics on job runs. The number of concurrent machines blows up because jobs will keep piling up if avg job takes 5 mins, need to be able to schedule multiple jobs on a machine. To make machine utilisation efficient and increase throughput.

**Interviewer:** "Blows up" isn't a number — 55,000 × 300, give me the figure. And reconcile: your own FR said each job runs on a fresh machine. Why does GitHub give every job a fresh machine, and does "schedule multiple jobs per machine" survive that reason?

**Aayush:** 16500000 machines.

**Interviewer:** Two follow-ups: (1) Why fresh machines — what breaks if job A and job B from different repos share a machine? (2) Do you believe 16.5M? Which input assumption would you revisit, and what's the realistic number?

**Aayush:**
1. Fresh machines needed so that there is isolation between the 2 jobs.
2. The job for each event assumption seems to be need of improvement.

**Interviewer:** On the contrary — one workflow file usually fans out to several jobs (a matrix of OS × language version), so jobs go up, not down. The lever you should pull is the traffic model itself. Give me the number you're going to design to.

**Aayush:** Help me out with this number.

**Interviewer:** I'm not going to hand you the number — here's the sanity check: your model has 100M daily active users each pushing 10 commits a day. Roughly how many professional software engineers exist in the world, and how many push code on a given day? Anchor on that, then re-derive.

**Aayush:** NFRs ->
1. Low latency job scheduling post event(<1s)
2. No event should be dropped.
3. Highly available for event receiving.
4. Atleast once guarantee for job execution.
5. Durably store artifacts for 7 days.
6. Assume 1M DAU, each user opens 1 PR a day, 10 commits/day -> 10 PR events/s, 100 commits/s. At peak 5x. At peak 550 jobs scheduled/s. Assuming each job generates 1Mb of artifacts -> 0.5GB/s of artifact data generated.
7. Execute jobs in fresh machine for isolation. Assuming each job 5 mins, need to schedule 550*300 = 165000 machines at peak.

**Interviewer:** Good — 165k concurrent VMs at peak is a number a real system is built around. Requirements locked. Rendering to canvas. Next: core entities — main objects, important fields, identifiers, relationships.

**Aayush:** Core Entities ->
1. Job (id, eventTriggerType, dependsOn:[JobId], createdAt, codeToExecute:string, executionEnvironmentDetails)
2. JobExecutionRecord (jobId, executionTime, artifactStoreUrl, logFileUrl, status: PASS|FAIL)
3.

**Interviewer:** (1) Is `Job` the definition parsed out of the YAML, or an instance of one run? If I push two commits to the same branch, how many Job rows exist and what distinguishes them? (2) A workflow has many jobs — what object does the single PR check status hang off, and is it in your list? Finish from #3.

**Aayush:** Core Entities ->
1. Job (id, dependsOn:[JobId], createdAt, codeToExecute:string, executionEnvironmentDetails)
2. JobExecutionRecord (jobId, executionTime, artifactStoreUrl, logFileUrl, status: PASS|FAIL)
3. Workflow(id, status, eventTriggerType, jobDag:[JobId])
4. EventWatcher (eventType, createdAt)

**Interviewer:** (1) Same instance-vs-definition problem now on Workflow — it has jobDag (definition) and status (per-run). If the same workflow runs 500 times a day, one row or 500? (2) You have nothing representing the machine. 165k of them at peak. What's that entity?

**Aayush:**
1. There will be a workflow object and job objects which will be created when a YAML file is parsed. Now when the actual events come in then executionRecords will be created for the workflow and job. Job will reference Workflow and JobExecutionRecord will reference WorkflowExecutionRecord.
2. Machine should not need an entity right?

**Interviewer:** At any moment you have 165k machines running jobs. A machine goes silent — process crashed, host died. How do you detect that and re-run the job, if nothing records which machine executes which job and when you last heard from it? And how do you route a job to a self-hosted runner? Decide: entity or not, and why.

**Aayush:** MachineStatus (lastPingTime:, alive, osImage)

**Interviewer:** Entities on canvas. Next: API design — endpoints with verbs, paths, explicit request/response shapes. Both what the browser calls (viewing runs, live logs, re-running, cancelling, downloading artifacts) and what the runner machine calls to the control plane. Include pagination and idempotency.

**Aayush:** API Design ->
NOTE: Identity extracted from Auth headers
1. GET /workflows/:id?cursor={}&limit={}
   Result: WorkflowExecutionRecord (status, jobs:[JobExecRecord])[], nextCursor. Cursor based pagination.
2. GET /workflows/:id/execution/:execRecordId
   Result: WorkflowExecRecord()
3. GET /workflows/:id/artifacts
   Result: ArtifactStoreUrls

The parsing of YAML and subscribing to new github events is done internally by a job which could execute on commit in which YAML files are changed or it could be periodic. There would be an event pushing webhook we would need to expose which would need to be registered with Github.

**Interviewer:** Three of those are read endpoints; the interesting half is missing. (1) Runner-facing contract: how does a machine get a job, say it's alive, report progress, report the final result? (2) Live logs — FR6 promises <500ms to the browser and none of your endpoints does that. What endpoint, what protocol, and why over the alternatives?

**Aayush:** Need help.

**Interviewer:** I'll give you the fork, not the answer. Runner: push or pull? Some runners are self-hosted behind a customer firewall. Pick a direction and defend it. Logs: name the transport options a browser has for a continuous server-side stream — you need one-directional server→client, surviving a page refresh mid-job. Which one, and what happens to the user who opens the page after the job finished?

**Aayush:** I quit.

**Interviewer:** Understood — stopping there. [Debrief delivered.]

---

## Design Summary

**Requirements Gathered:**
- 8 FRs covering YAML parsing → event watchers, scheduling on fresh machines, DAG dependency resolution, compute provisioning, sequenced execution, live log streaming, artifact storage, status reporting to commit/PR.
- NFRs: <1s scheduling latency, no dropped events, HA event receiving, at-least-once job execution, 7-day artifact durability, <500ms log latency, fresh-machine isolation.
- Scale (after one full revision): 1M DAU → ~110 events/s avg, 550 jobs/s at 5× peak, 0.5 GB/s artifacts, **165,000 concurrent machines at peak**.
- Out of scope: analytics on job runs (only item named).

**High-Level Architecture:**
None produced. The round ended during API design; no components, datastores, queues, or data flows were ever described.

**Key Design Decisions & Trade-offs:**
- Split definition entities (Workflow, Job, EventWatcher) from instance entities (WorkflowExecutionRecord, JobExecutionRecord) — arrived at after one prompt.
- Decided a machine-tracking entity was needed (MachineStatus with lastPingTime/alive/osImage) only after being pushed on crash detection.
- Cursor-based pagination on the workflow list endpoint.
- No named alternative was offered for any decision all round.

**Scalability & Fault Tolerance Points:**
- Computed 16.5M concurrent VMs under the original traffic model; first instinct was to relax his own fresh-machine isolation requirement rather than question the input. Revised the traffic model to 1M DAU only after being handed the sanity-check anchor.
- Machine liveness via heartbeat/lastPingTime — the one operability point landed, and only under direct prompting.

**Gaps / Missed Areas:**
- No high-level design at all; no deep dive.
- No runner-facing API (pull vs push never resolved); no live-log endpoint or transport choice; no re-run, cancel, or artifact upload endpoints.
- No response field shapes ("Result: WorkflowExecRecord()").
- No idempotency/dedupe on the webhook he exposed, despite GitHub webhooks being at-least-once and despite his own at-least-once NFR.
- Never noticed that "fresh VM" (30–60s boot) contradicts his own "<1s scheduling" NFR — the central design tension of the problem.
- Fan-out (matrix builds: one event → N jobs) never recognised; he asserted 1 event = 1 job and, when challenged, moved the estimate the wrong way.
- Long-tailed job duration never considered (p99 hours vs mean minutes).
- No hot-repo/tenant-fairness concern, no queue-depth or lag monitoring, no fleet cost, no job timeout.
- Did not reach for the Durable Task Framework experience on a workflow-orchestration problem.
- Three explicit requests for help, two of them on the round's highest-value questions.

---

## Feedback Given

# Debrief — GitHub Actions (Hard)

## 0. Pace report

| Phase | Reference | Actual (cumulative) | Delta | On pace? |
|---|---|---|---|---|
| Requirements | 8 min | 38 min | +30 | No |
| Core entities | 12 min | 63 min | +51 | No |
| API design | 17 min | 76 min (incomplete) | +59 | No |
| High-level design | 27 min | **never reached** | — | No |
| Deep dive | 40 min | **never reached** | — | No |
| Wrap-up | 45 min | 88 min (quit) | +43 | No |
| **Total** | 45 min | 88 min | +43 | No |

**Would this have fit a real 45-minute round? No — and not narrowly.** At the 45-minute mark you were roughly halfway through core entities. A real interviewer would have cut you off at requirements: you spent 38 minutes on a phase that is supposed to take 8, and most of that went on a traffic model you then discarded and rebuilt from scratch. You would have ended the interview having drawn zero boxes. There is no version of this round that reaches an architecture, let alone a deep dive.

**Biggest time sink:** the scale derivation. Not the arithmetic — the arithmetic was fine once you did it. The sink was that you asserted `100M DAU × 10 commits/day` without sanity-checking it, defended it for four exchanges, needed me to hand you the anchor ("how many engineers exist?"), and then re-derived the whole NFR block. Twenty-five minutes to arrive at a number you could have sanity-checked in thirty seconds by asking "is 1B commits a day plausible?"

## 1. Senior-signal scorecard

| Signal | Read | Why |
|---|---|---|
| **Owns the narrative** | **Weak** | Three explicit "help me out" / "need help" requests, including on the two highest-value questions of the round (runner pull-vs-push, log transport). Every trap in this problem was surfaced by me. |
| **Leads with trade-offs vs alternatives** | **Weak** | Not a single named alternative all round. No datastore named, no queue named, no transport named, no isolation mechanism named (VM vs container vs Firecracker). |
| **Pushes scale until it breaks** | **Weak** | The scale did break — 16.5M concurrent VMs — but you didn't push it there, I did, and when it broke your first instinct was to weaken your own isolation requirement rather than question the input. Then you fixed it by shrinking the problem 100× rather than designing for the break. |
| **API as a designed contract** | **Weak** | Three read endpoints, no response field lists (`Result: WorkflowExecRecord()`), no runner-facing API at all, no re-run, no cancel, no log stream, no idempotency on the webhook you exposed. Cursor pagination was the one real point scored. |
| **Operability / second-order** | **Mixed** | You did land the machine-liveness question once I forced it — `lastPingTime` + `alive` is the right instinct. Nothing else: no lag, no queue depth, no hot repo, no cost of a 165k fleet. |
| **Pace** | **Weak** | 88 minutes, no architecture. |

**Overall read: below mid-level *for this round*.** No-hire. That's not a read on your ceiling — your entity modelling was genuinely decent and the definition-vs-instance split you made after one nudge is a real modelling instinct. But a system design round where no system gets designed is a no-hire regardless of what came before it.

## 2. Performance Rating: **1/5**

No coherent architecture was produced. Requirements and entities existed; everything downstream of them did not.

## 3. What a senior strong-hire does on *this* problem

**Requirements, in 6 minutes.** "GitHub-scale: call it ~100k active repos pushing ~1M workflow runs/day, ~3 jobs per run via matrix fan-out, so ~3M jobs/day ≈ 35/s average, 5× peak ≈ 175/s. Average job 5 min → ~50k concurrent runners at peak. Job durations are long-tailed: p50 2 min, p99 an hour, and that tail is what makes scheduling hard." Note the two things you never said: **fan-out is the point** (one event → N jobs, that's what a matrix build *is*), and **duration is long-tailed, not average** — a mean of 5 minutes hides the 6-hour job that pins a machine.

**The trap you should have self-raised:** this is a **multi-step process + long-running task** problem, and you have the single best war story in the room for it — D365 Durable Task Framework. A workflow DAG with per-job retries, at-least-once execution, and durable state across hours-long jobs *is* a durable orchestrator. You never said the word. That's a free senior signal you left on the table for the second round running.

**Idempotency — the trap I withheld and you never touched.** GitHub webhooks are at-least-once. You exposed a webhook endpoint and gave it no delivery ID, no dedupe. A duplicate push event doubles the fleet cost and posts two conflicting check statuses on the same commit. A senior raises this unprompted: dedupe on `(repo, commit_sha, workflow_id, delivery_id)` with a unique constraint, and the second delivery is a no-op returning the existing run ID.

**The runner contract — the question you quit on.** Answer: **pull, not push.** Self-hosted runners live behind customer firewalls with no inbound connectivity, so the control plane can never dial out to them. Runners long-poll `POST /runners/{id}/acquire` with their labels (`ubuntu-22.04`, `self-hosted`, `gpu`), the scheduler hands back a job + a short-lived scoped token, and the runner heartbeats `POST /jobs/{runId}/heartbeat` every ~10s. Miss 3 heartbeats → the job is orphaned, requeued with `attempt++`. The trade-off you give up: pull adds a poll-interval of scheduling latency against your own <1s NFR, which you fix with long-polling rather than tight polling. That's the shape of a trade-off answer — choice, named alternative, what you sacrifice, how you mitigate.

**Logs — the other question you quit on.** Three options: polling, **SSE**, WebSockets. It's server→client only, so WebSockets buys you a bidirectional channel you don't need and costs you a stateful connection per viewer. SSE, with the runner batching lines every ~250ms to blob storage + a pub/sub fanout. The part almost nobody gets and a strong-hire does: **the user who opens the page after the job finished** — live logs and archived logs are two different systems wearing one URL, so the endpoint returns the sealed blob for completed runs and an SSE stream for in-flight ones, and a viewer joining mid-run needs the backfill-then-subscribe handoff without dropping or duplicating lines at the seam.

**Where the scale actually breaks — and it isn't the machine count.** 50k VMs is a procurement problem, not a design problem; you buy them. The break is **VM boot latency versus your <1s scheduling NFR**. A fresh, isolated VM takes 30–60 seconds to boot. Your NFR is unmeetable by construction. A senior catches this contradiction the moment they write "fresh machine" next to "<1s" and resolves it: a **warm pool** of pre-booted VMs per OS image, sized by forecast demand, with jobs assigned to already-running machines in milliseconds — plus a cold path when the pool drains. Now you own a genuinely hard second-order problem: pool sizing against a spiky, long-tailed workload, and the cost of idle warm capacity. *That* is the deep-dive conversation this problem exists to have, and it was thirty minutes past your reach.

**Operability you never touched:** the **hot repo** (one monorepo with a 200-job matrix starves every other customer — you need per-org fairness/quota in the scheduler, not FIFO); queue depth as your primary alert; the 6-hour job that never dies and needs a hard timeout; artifact storage cost at 7-day retention; and "how do I know logs are lagging" — end-to-end log latency measured with a synthetic canary job, not by trusting the pipeline.

## 4. Next step

Re-read `system_design_senior_guidance.md` — specifically the **pre-round self-check** at the top and the **eight patterns** table. This problem was "managing long-running tasks" + "multi-step processes," which the table tells you to answer with enqueue → worker pool → retries → DLQ → status endpoint. Naming the pattern in minute three would have given you the entire skeleton before you drew anything.
