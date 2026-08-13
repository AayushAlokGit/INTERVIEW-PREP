# LLD Sprint Transcript (scoping, timeboxed)
**Date:** 2026-08-13
**Start Time:** 19:05:13 · **End Time:** 19:15:13
**Problem:** In-process pub/sub message bus — publishers send messages on topics, subscribers receive them
**Category:** infrastructure object (concurrency-first)
**Difficulty:** 4/5 — per-subscriber ordering and at-least-once under many threads
**Scoping readiness: 2/5**
**Complete inside 10:00: yes** — single submission landed at 9:56; Out of Scope list never written, handler-failure rule left dangling
**Out of Scope list produced:** Never
**Orchestrator named:** Yes (`MessageBrokerSystem`)
**Entity revision passes:** 0

| Phase | Pace target | Landed at | ± vs target | Messages used | Score |
|---|---|---|---|---|---|
| Requirements | 6:00 | 9:56 | +3:56 | 1 | 2/5 |
| Entities & relationships | 10:00 | 9:56 | −0:04 | 1 | 4/5 |

## Walk coverage
| # | Item | Hit/Partial/Miss | Evidence |
|---|---|---|---|
| 1 | Actors & entry point | Hit | Producers, subscribers, `MessageBrokerSystem` as the entry object |
| 2 | Core operations | Hit | Publish, subscribe, async deliver, async cleanup to free capacity |
| 3 | Rules & legality | Hit | Full topic → exception; unknown topic → exception; at-least-once to every subscriber in publish order |
| 4 | Lifecycle & terminal states | Partial | Message lifecycle (published → delivered to all → cleaned up) stated; unsubscribe never raised, topic lifecycle never raised |
| 5 | Failure behaviour | Hit | "Throw exception" convention held across both failure paths |
| 6 | Multiplicity & domain variants | Partial | Many subscribers per topic with per-subscription cursor; never probed unsubscribe or fanout-vs-competing-consumers |
| 7 | Concurrency posture | Hit | Asked unprompted (question 4) and written as requirement 7: in-memory, multi-threaded, multiple producer and consumer threads |
| 8 | Explicit Out of Scope | Miss | None produced — despite having asked about durability and receiving "no", which was an Out of Scope line already handed to him |

**Budget allocation:** 3:30 on seven clarifying questions (guarantees, ordering, threading, sync/async, capacity, durability, subscriber-side execution) · 6:26 composing requirements + entities in a single message landing at 9:56 · zero slack.

**First-pass completeness:** One message carried both phases with no back-fill and no revision. This is the strongest first pass in the file so far. The cost was that composing it consumed every remaining second, leaving nothing for item 8.

**Silent assumptions:** (a) Overflow behaviour — I explicitly deferred it to him and he chose throw-at-publish, which is a resolution, not an assumption. (b) Subscribers see only messages published after they subscribe — never stated. (c) Handlers are invoked on a bus-owned thread rather than the subscriber's own — never stated. (d) Locks were placed on `Topic` and `Subscription` only *after* the buzzer; inside the box the concurrency requirement had no enforcement location.

**Dangling rules:** At-least-once delivery plus subscriber-side callback execution (his own question), with nothing said about what happens when a handler throws — at-least-once is precisely the guarantee that forces a retry/redelivery rule. Raised and left unresolved.

**Answers overridden after asking:** Two instances. Told topics are created on first use, he wrote "if no topic exists throw exception." Told the bound is a per-subscriber queue, he modelled a bounded per-topic log. Both are defensible designs, but overriding an answer you asked for without saying you are doing so reads as not listening.

## What he produced (verbatim, as it stood at 10:00)

### Clarifying questions (3:30)
1. How will subscribers subscribe to a topic?
2. What guarantees of delivery do we need to provide?
3. How does message need to be delivered? In what order and to what all subscribers?
4. Is this multi threaded?
5. Should message delivery be synchronous or asynchronous?
6. Is there any capacity constraint on the topics? If yes, how do we manage capacity in case of overflow?
7. Do we need to durably store the messages?
8. (with submission) Do we need to support executing subscriber side logic as well?

### Requirements
```
1. Producers can push messages to topic with capacity constraints , if topic is full then throw exception. If no topic exists throw exception.
2. Subscribers can subscribe to a topic and receive messages pushed to topic.If no topic exists throw exception.
3. Subscriber specific code triggerred on messages.
4. The messages must be delivered atlast once to every subscriber in the publish order.
5. Messages delivered asynchronously to subscribers.
6. Messages delivered to all susbcribers will be cleaned up asynchronously to free up capacity.
7. In memory , multi threaded i.e multiple producer and consumer threads.
```

### Out of Scope
_(none produced)_

### Entities & relationships
```
1. MessageBrokerSystem (Map<name,Topic>)
2. Topic(Name, List<Messages> , List<Subscriptions>, nextMsgId)
3. Message(msgId, payload)
4. Publisher
5. Subscriber( List<Subscription>)
6. Subscription (subscriber, topic, nextMsgId)
```

### Post-buzzer (not scored)
"Topic and Subscription will have locks." — correct placement, and the half of item 7 that was missing inside the box.

## What was still missing at 10:00
- Any Out of Scope list (hard cap: requirements max 2), even though the durability answer had already handed him one line of it.
- The handler-failure / redelivery rule implied by at-least-once (second cap, max 3).
- Unsubscribe, and subscription as a lifecycle with a terminal state.
- Where the locks live — stated forty seconds after the buzzer.
- `Publisher` is an entity with no state and no rule; `Subscriber.List<Subscription>` duplicates `Topic.List<Subscription>` with no owning direction named.

## Where the time went
3:30 on questions was a good investment and the best question batch in this file — item 7 was asked unprompted, which is a direct improvement on the recurring weakness. The remaining 6:26 was composition, not discovery: the requirement list arrived ordered, numbered and non-overlapping on the first attempt, so the time was spent writing well rather than working things out. That is the good failure mode, but it left literally four seconds of slack and no room for item 8.

## Ideal front half (writable in the same 10 minutes)

### Requirements
**Out of Scope:** persistence/durability across restart · cross-process or network transport · dead-letter queues · topic deletion · consumer groups / competing consumers · publisher backpressure.

1. Actors: publisher threads, subscriber threads, the bus. Entry point: `MessageBus`.
2. Ops: `publish(topic, payload)` (returns immediately), `subscribe(topic, handler)`, `unsubscribe(subscriptionId)`.
3. Rules: per-topic publish order preserved per subscriber; every active subscriber sees every message published after it subscribed; bounded buffer per subscriber — on overflow, throw at publish.
4. Lifecycle: message `PUBLISHED → DELIVERED(sub) → ACKED(sub)`, reclaimed once every subscription has passed it. Subscription `ACTIVE → CANCELLED` (terminal).
5. Failure: throw on unknown topic, full buffer, duplicate subscribe. Handler throws → retry N times, then drop and continue — at-least-once forces this rule to be stated.
6. Multiplicity: many topics, many subscribers per topic, fanout (not competing consumers); each subscriber holds an independent cursor.
7. Concurrency: multiple publisher and subscriber threads; per-topic lock on append, per-subscription delivery on its own worker so one slow subscriber cannot block the topic.
8. (Out of Scope as stated above.)

### Entities & relationships
- `MessageBus` — **orchestrator**; owns `Map<name, Topic>`
- `Topic` → name, bounded `List<Message>`, `List<Subscription>`, nextSeqId
- `Subscription` → id, topic, handler, cursor, state — **exists only to hold a rule**: requirement 3's per-subscriber ordering and requirement 5's retry count have nowhere else to live. He found this one on the first pass.
- `Message` → seqId, payload
- No `Publisher` class — publishing is a call, not an object.

**What this buys — requirements:** the handler-failure rule his at-least-once guarantee demands, unsubscribe as a terminal state, and an Out of Scope list one line of which the interviewer had already given him.
**What this buys — entities:** removes the one entity holding no state and enforcing no rule, and names a single ownership direction instead of `Topic` and `Subscriber` both holding subscription lists.

## Feedback given
- Best question batch in the file: seven questions covering guarantees, ordering, threading, sync/async, capacity and durability. Item 7 asked unprompted, then written down as a numbered requirement — a direct fix to a standing weakness.
- `Subscription(subscriber, topic, nextMsgId)` is exactly the rule-holding entity this drill exists to elicit, found on the first pass with zero revisions.
- Scored 2/5 on requirements anyway: no Out of Scope list, and the at-least-once guarantee was raised without its redelivery rule.
- Two answers were asked for and then silently overridden (topic auto-creation, per-subscriber bound).
- Lock placement was correct but arrived after the buzzer.
