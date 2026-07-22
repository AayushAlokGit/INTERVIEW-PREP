# S004 — Building Regression Coverage from Ambiguity (LLM Agent E2E Testing)
**Strength: 4.5 — Lead story**
**Use for:** Ambiguity / undefined problems, ownership & initiative, technical decision-making, cross-functional collaboration, GenAI / LLM depth, testing & quality, feature you owned, decomposition

---

## Spoken Delivery (say this version out loud)

I was a mid-level engineer on a team shipping an LLM-based sales opportunity agent on a weekly release cycle. The ask I got was deliberately vague: build a regression-catching mechanism before each release. No spec, no infrastructure guidance, no defined scope. And the stakes were real — if a regression slipped through, users would hit broken UI, like LLM-generated insights not rendering correctly.

My first move was to resist the urge to just start writing tests. I needed to understand what "regression coverage" actually meant for this feature, in reality and not in the abstract. So I went directly to the manual QA vendors who owned the existing process. I sat with them, mapped out every critical user flow they were walking through, and understood how they recorded observations. That grounded me in what actually mattered instead of my assumptions about it.

From there I hit the key architectural decision. The feature is built on an LLM, which is nondeterministic — so I couldn't apply standard assertion-based testing uniformly. I made a deliberate call to separate concerns. LLM *output quality* — is the insight good — would stay with a dedicated eval system. My E2E tests would own the *deterministic* layer: whether the agent could be set up, whether outputs rendered in the UI, and whether users could actually access them. I drew a pragmatic line for edge cases too — if an LLM-adjacent output was deterministic enough, like a date appearing in a specific format, I asserted on it rather than deferring it to the eval system. Then I worked with PMs and feature-area owners to build a prioritized list of P0 flows, going beyond what the QA vendors had been covering.

Honestly, the coordination was a challenge in itself. I had to align my schedule against a lot of busy calendars and repeatedly ping the PMs to get meeting time, but that stakeholder alignment was what made the priority list credible rather than my guess at it.

For the framework I chose Playwright, because it simulates real browser interactions — which mirrored exactly what the manual process had been doing. I wired it into our Azure DevOps pipelines and wrote custom pipeline code to trigger the suite, capture pass/fail results, and automatically distribute the report to stakeholders before each release. And because the feature was still actively being built, I designed the system to be extensible from the start, so new flows could be added easily as they shipped.

The result was that the system automated coverage of ten-plus critical user flows that had previously required manual execution. It caught multiple UI regressions before they reached users — ones that would have shipped otherwise. And it cut manual QA effort from one to two people spending at least a full day per release cycle down to an automated pipeline that ran without intervention. I also documented the pattern for adding tests to the LLM feature, and a new college grad used that doc to ramp up and start adding tests on his own.

The biggest thing I took away was that decomposing the problem space *before* writing a single line of code is the most important step when facing ambiguity. Separating the nondeterministic components from the deterministic ones early changed everything about how I approached the solution. I now do that decomposition explicitly on any ambiguous technical problem before I start building.

---

## The Story (STAR)

**Situation:**
I was a mid-level engineer on a team shipping an LLM-based sales opportunity agent on a weekly release cycle. I was asked to build a regression-catching mechanism before each release — but the ask was vague: no spec, no infrastructure guidance, no defined scope. The stakes were real: if regressions slipped through, users would hit broken UI, like LLM-generated insights not rendering correctly.

**Task:**
Turn an undefined "catch regressions before release" ask into a concrete, automated system — while owning the scope, the architecture, and the stakeholder alignment myself.

**Action:**
I resisted the urge to start writing tests and first went to understand what regression coverage meant in practice. I sat with the manual QA vendors who owned the existing process, mapped every critical user flow they walked through, and learned how they recorded observations — grounding the work in reality instead of assumptions. Then I made the key architectural call: because the feature is LLM-backed and nondeterministic, I separated concerns. LLM output *quality* stayed with a dedicated eval system; my E2E tests owned the *deterministic* layer — agent setup, UI rendering, and user access. For edge cases, I drew a pragmatic line: if an LLM-adjacent output was deterministic enough (e.g., a date in a specific format), I asserted on it rather than deferring it. I collaborated with PMs and feature-area owners — which took persistent calendar-wrangling — to build a prioritized P0 flow list that went beyond the vendors' coverage. I chose Playwright because it simulates real browser interactions, mirroring the manual process, wired it into Azure DevOps with custom pipeline code to trigger the suite, capture pass/fail, and auto-distribute the report to stakeholders before each release. I designed it to be extensible from day one since the feature was still evolving, and documented the pattern for adding new tests.

**Result:**
Automated coverage of 10+ critical user flows that previously required manual execution. Caught multiple UI regressions before they reached users — ones that would otherwise have shipped. Reduced manual QA from 1–2 people spending at least a full day per release cycle to a hands-off automated pipeline. My "how to add tests" documentation let a new college grad ramp up and contribute tests independently.

---

## Earned Insight
> "When the problem is ambiguous, the most important step happens before you write a line of code: decompose the space. For an LLM feature, the decisive cut was separating the nondeterministic part — output quality, which belongs to an eval system — from the deterministic part — setup, rendering, access, which E2E tests can assert on reliably. Once that line was drawn, every other decision got easy."

Say this out loud. It reframes the story from "I wrote some tests" to "I imposed structure on an undefined problem."

---

## Technical Depth Questions (Interviewer Voice)

- "Why can't you test an LLM feature the same way you'd test anything else? Walk me through the core problem."
  **Ans:** Because the LLM's output is nondeterministic — the same input can produce different phrasing, ordering, or content across runs. Standard assertion-based testing assumes a deterministic expected value, so if I assert on the exact LLM text, the test is flaky by construction and everyone learns to ignore it. The insight was that a single feature has two very different kinds of behavior mixed together: the *quality* of the generated content, which is nondeterministic, and the *plumbing* around it — can the agent be set up, does the output render in the UI, can the user access it — which is fully deterministic. You have to test those with different tools.

- "How exactly did you split responsibilities between the eval system and your E2E tests?"
  **Ans:** The eval system owned output quality — is the generated insight correct, relevant, useful. That's a judgment problem, often scored by an LLM-as-judge or human rubric, and it tolerates nondeterminism. My E2E suite owned the deterministic contract: the agent initializes correctly, the output actually renders in the UI without breaking, and the user can access it through the real flow. So the eval answers 'is the insight good,' and my tests answer 'did the feature actually work end to end and show up on screen.' Keeping those separate meant my tests were stable and trustworthy, and quality regressions had a dedicated home instead of leaking into flaky E2E assertions.

- "How did you handle the gray-area outputs — things derived from the LLM but not free-form text?"
  **Ans:** I drew a pragmatic line rather than a dogmatic one. If an LLM-adjacent output was deterministic enough to assert on reliably — for example a date rendered in a specific format, or a field that always follows a fixed structure — I included that assertion in the E2E suite instead of punting it to the eval system. The test is 'is this specific deterministic property correct,' not 'is the LLM's prose good.' The principle was: assert on anything with a stable expected value; defer only the genuinely nondeterministic quality judgments.

- "Why Playwright specifically? Did you consider alternatives?"
  **Ans:** The manual QA process was people driving a real browser through real user flows. Playwright mirrors that exactly — it simulates real browser interactions, so my automated tests exercised the feature the same way a user (and the manual vendors) did, including the UI rendering that was the actual failure mode. That fidelity to the real user path was the deciding factor: the regressions we cared about were UI-rendering and access issues, and a real-browser tool catches those where a lower-level API test wouldn't.

- "How did you integrate it into the release process — what did the pipeline actually do?"
  **Ans:** I wired the suite into our Azure DevOps pipelines and wrote custom pipeline code to do three things: trigger the suite as part of the pre-release flow, capture the pass/fail results, and automatically distribute a report to stakeholders before each release. So the regression signal arrived where and when the release decision was being made, without anyone having to run anything manually. That last part — auto-distributing the report — is what turned it from 'a test suite exists' into 'the release process consumes the signal.'

- "You said you designed it to be extensible — what did that mean concretely?"
  **Ans:** The feature was still actively being built, so new user flows were going to keep appearing. I structured the suite so adding a new flow was a well-defined, low-friction operation rather than a rewrite — consistent patterns for setup, interaction, and assertion that a new test could follow. I also documented that pattern explicitly. The proof it worked: a new college grad picked up the doc and started adding tests for the feature on his own, without me hand-holding each one.

- "How do you keep a Playwright suite against a live, evolving feature from becoming flaky?"
  **Ans:** Two ways, and both trace back to the core decision. First, I kept nondeterministic quality assertions out of the E2E suite entirely — those went to the eval system — so the E2E tests only assert on things with stable expected values, which removes the biggest source of flakiness. Second, real-browser tests still need disciplined waiting on actual UI state rather than fixed timeouts, and consistent, isolated setup per test so flows don't interfere. The architectural split is what made the rest tractable — a suite that only asserts deterministic properties is inherently far more stable.

---

## Behavioral Follow-Up Questions (Interviewer Voice)

- "The ask was vague — no spec, no scope. Where did you even start?"
  **Ans:** I deliberately didn't start by writing tests, which was the tempting move. I started by getting grounded in reality: I went to the manual QA vendors who already owned the existing process and sat with them to map every critical flow they walked through and how they recorded results. That gave me a concrete picture of what 'regression coverage' actually meant for this feature instead of me inventing a scope. From there I could make real decisions — the architecture split, the P0 prioritization — on a foundation of how the feature was actually used, not assumptions.

- "How did you decide what to cover first when everything could be a P0?"
  **Ans:** I didn't decide alone — that would just be my opinion. I took what I'd learned from the QA vendors and worked with the PMs and feature-area owners to build a prioritized P0 list. The people who own the feature and the customer relationship have the best sense of what breaking would hurt most. My job was to synthesize their input into a coverage plan, and deliberately go beyond what the vendors had been manually covering where we identified gaps.

- "You mentioned coordinating with stakeholders was hard. How did you handle that?"
  **Ans:** It was genuinely one of the harder parts — the PMs had packed calendars, so I had to align my schedule to theirs and repeatedly, politely, keep pinging to get meeting time. I treated it as part of the job rather than a blocker: the priority list was only credible because it came from those conversations, so getting the time was worth the persistence. The lesson was that on an ambiguous, cross-functional problem, chasing alignment is real work — not overhead — and I had to own driving it.

- "This was handed to you loosely — did anyone tell you to talk to the QA vendors or pick the architecture? Or did you drive that?"
  **Ans:** I drove it. The ask was just 'build something to catch regressions before release' — the decision to ground it in the vendors' existing process, the architectural split between eval and E2E, the framework choice, the pipeline integration, and the extensibility were all mine to figure out. That ownership is the part of this I'm most proud of: I was handed ambiguity and returned a system.

- "What was the hardest technical decision, and how did you make it?"
  **Ans:** Separating the nondeterministic from the deterministic — deciding what my E2E tests should and shouldn't try to assert. It was tempting to try to make the tests validate everything including output quality, but that path leads to flaky, ignored tests. I made the call to give quality to a dedicated eval system and have E2E own only the deterministic contract, with a pragmatic exception for deterministic-enough edge cases. That single decision is what made everything downstream stable and tractable.

- "How did you know the system was actually valuable and not just busywork?"
  **Ans:** Concrete outcomes. It automated 10-plus flows that used to be manual, it caught multiple UI regressions that would otherwise have shipped to users, and it cut manual QA from one to two people spending a full day-plus per release to a pipeline that ran with zero intervention. And it had a second-order payoff: a new grad ramped on the feature by adding tests using my documentation, so it accelerated the team, not just the release.

- "If you had more time or had to scale this up, what would you do next?"
  **Ans:** I'd deepen the integration between the two layers — feed the eval system's quality signals into the same pre-release report so stakeholders see one unified regression picture, deterministic and quality-based together. I'd also invest more in test data and environment isolation as the flow count grows, and formalize the 'add a new flow' pattern even further since the feature kept evolving. The foundation — the deterministic/nondeterministic split and the pipeline hook — was built to extend, so this is growth, not rework.

---

## Watch Out For

- **Lead with the decomposition, not the tooling.** The interviewer is testing how you handle ambiguity. The memorable move is separating nondeterministic (eval system) from deterministic (E2E) — Playwright and Azure DevOps are just implementation. Don't let it become "I set up Playwright."
- "Why not just test the LLM output directly?" → Explain flakiness-by-construction: asserting on nondeterministic text makes tests everyone learns to ignore. That's why quality goes to a dedicated eval system.
- Own the ambiguity as a positive. Don't frame "no spec, no scope" as a complaint — frame it as the thing you converted into a system through decomposition and stakeholder work.
- Have the numbers ready and precise: 10+ flows automated, multiple regressions caught pre-ship, 1–2 people × a full day per cycle eliminated, plus the new-grad-ramped-on-my-docs proof point.
- If asked about GenAI depth, this is your opening — mention the eval system / LLM-as-judge boundary and how you'd unify the signals. It pairs well with your MCP-server RAG talking point.
- Don't undersell the coordination struggle — but frame the persistent PM-pinging as ownership of alignment, not as a grievance about busy people.
