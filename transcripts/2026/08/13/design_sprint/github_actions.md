# Design Sprint Transcript (front half, timeboxed)
**Date:** 2026-08-13
**Start Time:** 17:22:16 · **End Time:** 17:39:11
**Problem:** GitHub Actions (CI/CD execution service)
**Front-half readiness: 3/5**
**Front half complete inside 17:00: no** — the API reached only one read endpoint. No trigger, cancel, re-run, log-stream or runner-registration endpoint existed at the buzzer.

| Phase | Pace target | Landed at | ± vs target | Messages used | Score |
|---|---|---|---|---|---|
| Requirements | 8:00 | 10:35 | +2:35 | 1 | 3/5 |
| Core entities | 12:00 | 14:35 | +2:35 | 1 | 4/5 |
| API design | 17:00 | 16:55 | on time | 1 | 1/5 |

**Budget allocation:** 10:35 requirements / 4:00 entities / 2:20 API. Requirements borrowed 2:35 and entities spent its full allowance without repaying any of it, so the API opened at 14:35 with 2:20 left for a phase needing six endpoints. **The API paid the entire bill for both earlier overruns**, and it is the phase an interviewer checks most cheaply.

**First-pass completeness:** One message per phase, no clarifying questions, no back-fill. Not a completeness problem — an allocation problem.

**Plausibility check:** Attempted in spirit, not executed. He derived "165,000 machines needed concurrently at peak" — a genuine capacity number, and exactly the kind of derivation previously missing — then stated it and moved on. Two problems one sentence would have caught: (a) the arithmetic double-applies peak (110 events/s × 5 min = 33,000 concurrent; the ×5 was already in the 110 or should not be applied again to a sustained-concurrency figure), and (b) 165k concurrent isolated VMs is a hyperscaler fleet — noticing that the number is implausible for the stated 1M DAU is the check being scored. Caps Requirements at 3.

## What he produced (verbatim, as it stood at 17:00)

### Requirements
```
FRs ->
1. Resolve and parse workflow YAML files (defined in repo) to generate a DAG of
   jobs for the workflow.
2. Register event listener for repository events like a commit or raising PR to
   trigger workflow execution.
3. Execute jobs on a pool of provisioned runners.
4. Workflow run (i.e the inner job runs) logs live streamed to users.

Out of scope ->
1. Storing artifacts produced during pipeline runs.
2. Retaining workflow run logs for long term.

NFRs ->
1. Highly available for accepting event triggers (99.99~52mins downtime/year)
2. The workflow executions might be long running and should be durable.
3. Upto 3 min latency for scheduled jobs to run.
4. Near real time log streaming , latency of 2s.
5. Assuming 1M DAU with each user raising 1 PR and pushing 10 commits a day ->
   10 PRs events/s + 100 commit events/s ~ 110 events/s avg , peak 5x.
   Assuming each workflow runs for 5 min avg -> 165,000 machines needed at peak
   concurrently.
6. Each workflow executes in isolated machine.
7. Workflow run data retained for 7 days.
```

### Core entities
```
1. WorkflowDefinition(repo, eventType, jobSequence:[jobId1, jobId2])
2. JobDefinition(workflowId, steps, dependsOn: [jobId..], envType:{})
3. WorkflowExecution(workflowId, executionTime, createdAt, status)
4. JobExecution(jobId, executionTime, createdAt, logS3FileLink, status)
5. ComputeMachine(machineType, os_image, status: IDLE | RUNNING_JOB, updatedAt)
```

### API design
```
1. GET /workflows/:id/workflowExecution/:id?cursor={}&limit={}
Response: WorkflowExecution(workflowId, executionTime, status,
  jobExecutions:{JobExecution(status, logS3FileLink)})
If workflow currently running returns live logs else returns stored logs
```

## What was still missing at 17:00

**Requirements:**
- Items 6 (isolation) and 7 (retention) are constraints/FRs wearing an NFR label; no consistency posture, no read:write ratio, no storage growth for logs (the dominant data volume in this system).
- The 165k-machine figure is internally inconsistent and unchecked.
- No throughput target for the log-ingest path despite a 2s streaming NFR.

**Core entities (strongest set of the sitting):**
- **No log chunk/segment entity.** `logS3FileLink` cannot serve an in-progress stream, so FR 4 and the 2s NFR have no data model behind them.
- No pending-job / queue entry entity, though scheduling onto a runner pool is FR 3.
- No `Runner` registration/heartbeat/lease fields on `ComputeMachine` — no `leaseExpiry`, no `currentJobId`.
- No keys or uniqueness constraints; `WorkflowDefinition` has no id.
- `jobSequence` as a flat list contradicts `dependsOn` as the DAG representation — two mechanisms for the same thing.

**API design (the phase that collapsed):**
- **No trigger endpoint** — nothing can start a workflow run.
- No cancel, no re-run of a failed job.
- **No log-streaming endpoint or transport**, despite it being FR 4 with a 2s target.
- No runner registration, heartbeat, or job-claim endpoint.
- No list endpoint for a repo's workflow executions.
- The one endpoint written has a malformed path — `/workflows/:id/workflowExecution/:id`, two identically named params.
- `cursor`/`limit` applied to a single-resource read, and no `nextCursor` in the response.
- "If running returns live logs else stored logs" is a behaviour note, not a contract — no shape, no transport, no status code.
- No idempotency on the (absent) trigger endpoint, which is the at-least-once ingress in this design.

## Where the time went
Requirements ran 2:35 over on volume — seven NFRs where four would do, two of which aren't NFRs. Entities ran a further 2:35 over, and produced the best entity set of the sitting, so the time bought something. But nothing was repaid, and the API opened with 2:20 left. He got one endpoint down. The failure is sequencing, not knowledge: the six missing endpoints are obvious ones he could have listed as bare verbs and paths in under a minute had he started the phase on the clock.

## Ideal front half (writable in the same 17 minutes)

### Requirements (~7 min)
**FRs:** repo event (push/PR) triggers a run · parse workflow YAML into a job DAG · schedule jobs onto isolated ephemeral runners respecting `dependsOn` · stream logs live · cancel/re-run.
**Out of scope:** artifact storage, caching, self-hosted runners, billing, matrix expansion.

**NFRs:**
- Availability 99.99% on event ingest — a dropped webhook is a build that silently never runs. **Durable enqueue before ack.**
- Consistency: a job must run **at most once** per (run, job, attempt) — double-executing a deploy is the failure that matters. Run status read-your-writes for the triggering user.
- Latency: queue → job start p95 < 30s, p99 < 3 min. Log line → viewer p95 < 2s.
- Scale: 1M DAU, ~10 pushes + 1 PR/day = 11M events/day ÷ 10^5 = **110 events/s avg, 550/s peak**.
- Concurrency: 110/s × 300s avg runtime = **33,000 concurrent runners** steady state; peak ~**165,000**.
- **Plausibility check:** "165k concurrent VMs at peak is a very large fleet — that's GitHub's actual order of magnitude, so it's plausible for this DAU, but it means warm-pool sizing is the whole problem. Most peaks are business-hours regional, so a global fleet smooths this; I'd provision ~50k warm and burst."
- Log volume: 33k concurrent jobs × ~1 KB/s ≈ **33 MB/s ingest**, ~2.8 TB/day; 7-day retention ≈ 20 TB. **Logs are the dominant storage, not run metadata.**
- Durability: run state survives a scheduler crash — a 40-minute job cannot be lost.

*What this buys:* the same arithmetic he did, plus the check. The check reveals that 165k is peak-not-steady and that warm-pool sizing is the real design problem — and adds the log-volume line, never computed despite logs being the biggest thing in the system.

### Core entities (~3 min)
- **WorkflowDefinition** — `id`, `repoId`, `path`, `commitSha`, `triggers[]`, `parsedDagJson`. Unique `(repoId, path, commitSha)` — definitions pinned to a commit, so re-running an old build uses the old YAML.
- **JobDefinition** — `id`, `workflowDefId`, `name`, `steps[]`, `dependsOn[]`, `runsOn`. (One DAG mechanism, not two — drop `jobSequence`.)
- **WorkflowRun** — `id`, `workflowDefId`, `repoId`, `triggerEventId`, `status: QUEUED|RUNNING|SUCCESS|FAILED|CANCELLED`, `startedAt`, `finishedAt`. Index `(repoId, createdAt desc)`.
- **JobRun** — `id`, `runId`, `jobDefId`, `status`, `attempt`, `runnerId`, `leaseExpiresAt`, `startedAt`. Unique `(runId, jobDefId, attempt)` — this is the at-most-once guarantee.
- **Runner** — `id`, `machineType`, `image`, `status: IDLE|LEASED|BUSY|DRAINING`, `currentJobRunId`, `lastHeartbeatAt`. *Load-bearing:* heartbeat + lease expiry are how a dead runner's job gets reclaimed rather than hanging forever.
- **LogSegment** — `jobRunId`, `seq`, `byteOffset`, `s3Key`, `sealed: bool`. *Load-bearing:* a live stream can't read a finished S3 object; the tail is served from hot storage, older segments from S3. **This is the entity that makes FR 4 possible.**

*What this buys:* his entity set was genuinely good — definition/execution split and a machine status enum. The additions are the lease (dead-runner recovery), the attempt counter (at-most-once), and the log segment (the 2s streaming NFR had no data model behind it).

### API design (~7 min) — **start this at 12:00 regardless of where entities are**
```
# --- ingest ---
POST /webhooks/{provider}                      Idempotency-Key: <delivery-id>
  req  { repoId, eventType, commitSha, ref, ... }
  res  202 { runIds: [...] }                   // ack only after durable enqueue
       // dedup on provider delivery id — webhooks are at-least-once

POST /repos/{repoId}/runs                      Idempotency-Key: <uuid>   // manual dispatch
  req  { workflowPath, ref, inputs{} }
  res  201 { runId, status: "QUEUED" }

# --- read ---
GET /repos/{repoId}/runs?status=&branch=&cursor=&limit=50
  res  200 { runs: [{ runId, status, commitSha, startedAt, durationMs }], nextCursor }

GET /runs/{runId}
  res  200 { runId, status, jobs: [{ jobRunId, name, status, dependsOn[], attempt }] }
       // no cursor: a run's job list is bounded and small

# --- logs ---
GET /jobs/{jobRunId}/logs?sinceSeq={n}         // poll / incremental
  res  200 { segments: [{ seq, lines[] }], nextSeq, sealed: bool }
GET /jobs/{jobRunId}/logs/stream               // SSE, live tail, ~2s target
       // same seq cursor as above, so a client can switch between them mid-job

# --- control ---
POST /runs/{runId}/cancel                      Idempotency-Key   -> 202
POST /runs/{runId}/rerun    req { failedOnly: bool }  Idem-Key   -> 201 { newRunId }

# --- runner control plane ---
POST /runners/register    req { machineType, image }   -> 201 { runnerId, leaseToken }
POST /runners/{id}/claim  req { labels[] }             -> 200 { jobRunId, steps[], leaseExpiresAt }
                                                       -> 204 no work
POST /runners/{id}/heartbeat  req { jobRunId }         -> 200 { leaseExpiresAt, shouldCancel: bool }
POST /jobs/{jobRunId}/complete  req { status, exitCode }  Idem-Key -> 200

Errors: 409 cancel on a finished run · 410 expired lease (runner must stop) · 429 concurrency limit
```

*What this buys:* this is the phase that scored 1/5, and the gap wasn't knowledge — it was opening the phase at 14:35. The endpoint list above, verbs and paths only, is about 60 seconds of typing. Write that skeleton first, then fill shapes downward until the buzzer; a complete list with thin shapes reads far better than one endpoint with a rich shape.

## Feedback given
The capacity derivation is real progress — he produced a concrete fleet-size number unprompted, which earlier sittings never did. The remaining half of that habit is testing it: both the arithmetic and the plausibility of 165k machines fail a one-line check. The API result is the headline problem. A CI system with no way to start a build would not survive the first minute of an HLD discussion, and it was lost to allocation rather than to any gap in understanding.
