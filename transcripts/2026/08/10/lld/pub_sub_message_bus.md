# LLD Round Transcript
**Date:** 2026-08-10
**Start Time:** 12:17:48 · **End Time:** 13:48:25 · **Duration:** 91 min
**Problem:** In-Process Pub/Sub Message Bus
**Category:** infrastructure object
**Performance Rating:** 2/5  <!-- machine-read on future rounds; <=3 = eligible for re-ask, >=4 retired -->
**Hints Used:** 0/2
**Requirements Asked:** topic creation/ownership · publish to non-existent topic · subscribe to non-existent topic · message limit per topic · delivery guarantee · payload type · topic name uniqueness · **Never Asked:** sync vs async delivery · duplicate subscription by same subscriber · fate of undelivered messages on unsubscribe · cross-topic ordering · whether the bus is used from multiple threads
**Out of Scope list produced:** Never
**Self-Verified:** No — trace had to be requested, and the claimed trace was wrong (returned a dependency, not state; missed that nothing calls cleanup)
**Concurrency follow-up:** correctness + coordination — named the correctness category before reaching for a lock, identified all three pieces of shared state, chose per-topic and per-subscription lock granularity, kept `onMessage` outside the lock, avoided deadlock by never holding both locks at once. Cost stated only after two pushes, and thinly.
**Would it have fit a real 40-min round?** No — cut off during class design; implementation, concurrency and both extensibility follow-ups would never have been reached.

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements + Out of Scope | 5 min | 8.0 min | +3.0 | Close |
| Entities & relationships | 8 min | 28.5 min | +20.5 | No |
| Class design | 20 min | 50.7 min | +30.7 | No |
| Implementation + trace | 32 min | 60.7 min | +28.7 | No |
| Extensibility + concurrency | 40 min | 91.0 min | +51.0 | No |
| **Total** | 40 min | **91 min** | **+51** | **No** |

---

## Problem Statement

Design an in-process publish/subscribe message bus. Publishers send messages to topics; subscribers receive messages from topics they've subscribed to.

Deliberately under-specified: no entity list, no rules, no scale, no concurrency hint. Single process. Pseudo-code chosen for phase 4.

---

## Conversation Log

**Interviewer:** Design an in-process publish/subscribe message bus. Publishers send messages to topics; subscribers receive messages from topics they've subscribed to. Single process — this is an object model, not a distributed system. Five phases with a reference timeline (measured, not enforced). Which language for phase 4? Any questions before you start?

**Aayush:** Here are what I believe the requirements should be:
1. Publishers can push messages to a topic
2. Subscribers can subscribe to a topic
3. Subscribers of a topic are delivered messages which have been pushed to the topic

Who will create the topics? What will happen when a publisher tries to push a message to a topic which does not exist? What will happen when a subscriber tries to subscribe to a topic which does not exist? Is there any limit to the messages that a topic can contain? What delivery guarantee must be provided?

**Interviewer:** Answers, committed for the round: (1) Topics are created explicitly through the bus; they do not spring into existence on first publish. (2) Publishing to a non-existent topic is an error the publisher must find out about. (3) Subscribing to a non-existent topic, same. (4) Yes — each topic is bounded, capacity configured at creation; what happens at capacity is your design decision, pick one and justify it. (5) At-least-once, and messages are delivered to each subscriber in publish order for a given topic. Delivery is asynchronous — publishing must not block on subscribers. Still owe me a language.

**Aayush:** pseudocode

**Interviewer:** Pseudo-code it is. Go ahead.

**Aayush:**
1. Publishers can push messages to a topic
2. Subscribers can subscribe to a topic
3. Subscribers of a topic are delivered messages which have been pushed to the topic
4. Topic creation independent of publish and subscribe
5. Publishing to non-existent topic throws exception
6. Subscribing to non-existent topic throws exception
7. Each topic is bounded by a capacity set while creating topic
8. At-least-once delivery guarantee for messages
9. Messages must be delivered in publish order

*(Requirements phase complete — 8.0 min. No Out of Scope list produced.)*

**Interviewer:** Requirements are on the sheet. Next: entities and relationships — what objects exist, who owns whom, and which one orchestrates.

**Aayush:** Will messages be simple string messages?

**Interviewer:** A message carries an arbitrary payload — treat it as an opaque value the bus doesn't inspect. Whether that's a string or something richer is yours to model.

**Aayush:** Are topics having unique names?

**Interviewer:** Yes — topic names are unique and identify the topic.

**Aayush:**
```
MessageBus (topics: Topic[], topicSubscribersMap)
Subscriber
Publisher (messageBus: MessageBus)
Message (status: DELIVERED | NOT DELIVERED)
Topic (name, capacity, messages: Message[])
```

**Interviewer:** Two questions. (1) `Message.status` — which requirement needs that field? Scenario: topic `orders` has three subscribers; the message has been handed to two of them and not the third. What does `status` read? (2) `Subscriber` has no state and no behaviour listed. What is it?

**Aayush:** The current list is not good enough, it does not cover many entities. Creating a new list:
```
MessageBus   (topics: Topic[])
Subscriber   (subscriptions: Subscription[])
Publisher    (messageBus: MessageBus)
Subscription (subscriber, topic)
Message      (payload, id)
Topic        (name, capacity, messages: Message[], subscriptions: Subscription[])
```

**Interviewer:** Taking the new list. A scenario against it: topic `orders` has capacity 5 and holds m1…m5. Subscriber A is fast and has processed all five; subscriber B is slow and has processed only m1 and m2. A publisher pushes m6. What does your model do with m6? When m1 is eventually removed from `Topic.messages`, what makes that safe? Answer with the fields you'd need, and which class holds them.

**Aayush:** m6 appended to `Topic.messages[]`, so topic.messages will have the messages in publish order. We can also store another attribute in subscription called `lastMessageSent`. In order to remove a message from the topic queue, we need to confirm it has been sent to all subscriptions, i.e. `lastMessageSent` of all subscriptions >= the message to be deleted.

**Interviewer:** Follow that through. Capacity is 5, the topic already holds m1…m5, and m1 cannot be removed because B hasn't received it. m6 arrives. What happens to m6, and what does the publisher see?

**Aayush:** m6 will be rejected by the current design.

**Interviewer:** Rejected how — exception, false return, silent drop? Commit to the contract. Then move to class design.

**Aayush:** Exception shown to publisher due to topic capacity being reached.

*(Entities phase complete — 28.5 min.)*

**Interviewer:** Entities and both rules are on the sheet. Now class design — every class, state with types, public methods with full signatures and return types.

**Aayush:** *(first class outline — MessageBus, Publisher, Subscriber, Subscription with `nextSequenceNumber`, Topic with `nextSequenceNumber`, Message with `sequenceNumber`; no return types on any method)* The nextSequenceNumber is load bearing because it helps determine which next message needs to be sent across the subscription channels. It must be a monotonically increasing integer which is set in the Topic class.

**Interviewer:** Three things, all in one turn: (1) Return types — almost none of those signatures have one. (2) Trace the delivery: I call `bus.subscribe("orders", mySubscriber)`, then a publisher publishes — walk me class by class through how the payload ends up executing my subscriber's code. Where does the subscriber's own logic live? (3) `Subscription.getNextSequenceNumber()` — who calls it, and what do they do with the value?

**Aayush:** *(full class outline with return types and a named exception per failure mode — see design sheet)*

2. When `bus.subscribe("orders", mySubscriber)` is called, MessageBus looks up the orders topic and creates a Subscription containing the topic, mySubscriber, and nextSequenceNumber = 1. The subscription is added to both the topic and subscriber. When the publisher calls `publish("orders", payload)`, MessageBus looks up the topic and delegates to `Topic.addMessage(payload)`. The topic creates a Message with the next sequence number, say 1, and adds it to its message queue. The topic then asynchronously triggers `deliverPendingMessages()`. The topic looks at each subscription and finds that this subscription's nextSequenceNumber is 1, so it selects message 1. The topic invokes the subscriber's callback, for example `subscription.subscriber.onMessage(message)`. The subscriber's implementation of `onMessage()` contains the subscriber-specific business logic. Once the subscriber successfully processes the message, it acknowledges it through the subscription. The subscription advances nextSequenceNumber to 2. If processing fails or the message isn't acknowledged, nextSequenceNumber remains 1, so the message can be delivered again. This gives us at-least-once delivery while the sequence number ensures messages are delivered in order.

3. It will be called by the async delivery mechanism to figure out which message to deliver across a subscription channel.

*(Class design complete — 50.7 min.)*

**Interviewer:** Clarification for the sheet: your trace calls `subscriber.onMessage(message)`, but `onMessage` isn't in your `Subscriber` class outline. Is `Subscriber` a concrete class users subclass, or an interface they implement? Then implementation: pseudo-code for `Topic.addMessage`, `Topic.deliverPendingMessages`, `Topic.cleanupDeliveredMessages`.

**Aayush:** Add a new method to subscriber to allow for execution of subscriber code on message delivery.

```
Topic.addMessage(payload):
    Generate a new monotonic sequence number
    Create a message object
    Append message object to messages list
    Trigger async delivery mechanism

Topic.deliverPendingMessages():
    Iterate through all subscriptions.
    For each subscription:
        1. Get nextSequenceNumber
        2. Find the corresponding Message
        3. If the message is available, call subscriber.onMessage(message)
        4. Subscriber acknowledges successful processing
        5. Subscription.acknowledge() advances nextSequenceNumber
    If the subscriber fails to process the message:
        - Do not advance nextSequenceNumber
        - The message will be retried later

Topic.cleanupDeliveredMessages():
    Find the minimum nextSequenceNumber across all subscriptions.
    Remove all messages from the messages queue whose sequenceNumber is less than that minimum.
    If there are no subscriptions:
        Do not clean up messages.
```

**Interviewer:** Now trace it. Topic `orders`, capacity 2. Subscriber A subscribes. A publisher publishes m1, m2, m3 in order, and A processes and acknowledges each successfully. Step by step through your three methods — after every step state `messages` contents, `topic.nextSequenceNumber`, and `subscription.nextSequenceNumber`. What happens at each publish, including the third?

**Aayush:** m1 and m2 publish succeed. The m3 publish success depends entirely on whether any of the previous messages have been cleaned up or not.

**Interviewer:** I want the values, not the dependency. Give me the three state variables after each publish. And in your `addMessage` pseudo-code as written — which of its four steps decides whether the publish succeeds?

**Aayush:** messages -> {m1,m2}; topic.nSN -> 2; subscription.nSN -> 1 when m1 not pushed, 2 when m1 pushed, 3 when m2 pushed. The appending to message list determines if publish succeeds depending on list size.

*(Implementation phase complete — 60.7 min. Claimed trace taken at face value; verified silently afterwards.)*

**Interviewer:** Concurrency. Two publisher threads call `bus.publish("orders", ...)` on the same topic at the same moment, while the async delivery mechanism runs `deliverPendingMessages()` on that topic, and a third thread is in `cleanupDeliveredMessages()`. What breaks? Name what goes wrong before you name a fix — then what you'd do and what it costs.

**Aayush:** There will be contention for the common resource here which would be the messages list. Also the sequence number allotted to the concurrent messages could turn out to be the same, so a correctness issue has been introduced due to concurrency. The publish order is broken, and now if even one of these messages is successfully delivered then there is a possibility that the cleanup will clean up both messages with one message not being delivered. This can be fixed by using locks. We need to protect the shared resources topic.nextSequenceNumber, topic.messages and also subscription.nextSequenceNumber. Subscription needs to lock to prevent multiple delivery threads delivering messages concurrently. So the topic lock will protect topic.sequenceNumber and topic.messages in the addMessage method. The subscription lock will protect subscription.sequenceNumber in the delivery path — only one delivery attempt across a subscription channel will advance the sequence number.

**Interviewer:** (1) What does the topic lock cost you at runtime — who waits on whom, and what if `onMessage` is slow? (2) `cleanupDeliveredMessages()` reads every subscription's cursor and mutates `topic.messages` — which locks does it take, in what order, and what risk have you created?

**Aayush:** 1. The topic lock costs publish throughput at runtime. In the delivery method the subscriber onMessage logic needs to be outside the lock; the topic lock should only be used to get the message to be delivered.

**Interviewer:** Good on keeping `onMessage` outside the lock. Still owe me #2 — lock ordering across cleanup and delivery.

**Aayush:** In the cleanup method we can have subscriptionLock to find the minSequenceNumber to be deleted, then release this lock and acquire topic lock to delete the message. This way the cleanup will only hold one lock at a time.

**Interviewer:** Extensibility follow-up 1. Some subscribers must receive the topic's entire retained history from the moment they subscribe, not just messages published after they joined. Others keep today's behaviour. Where does that land?

**Aayush:** What does topic history mean — the messages that have come in?

**Interviewer:** Whatever is still retained in `topic.messages` at the moment they subscribe.

**Aayush:** Can you give an example so I better understand?

**Interviewer:** Topic `orders` has had m1…m7 published; m1–m4 were cleaned up, the queue holds m5, m6, m7. Subscriber C subscribes now. Under your current design C receives only m8 onward. New requirement: C must receive m5, m6, m7 first, then m8 onward — while subscriber D, subscribing at the same instant, still gets only m8 onward.

**Aayush:** The nextSequenceNumber which is set at the time of creating a subscription should be decided based on the message list of the topic at the time of subscription creation.

**Interviewer:** Right seam. What's the change to the public API surface? Then follow-up 2: some topics now need competing-consumer semantics — a message goes to exactly one subscriber in the group, not all. What changes, and does anything have to be rewritten?

**Aayush:** An argument can be passed to `MessageBus.subscribe()` defining the behaviour. How would the subscriber to which this message is sent be determined?

**Interviewer:** Any subscriber in the group — round-robin is fine. The rule is only that exactly one gets it, and the message isn't cleaned up until that one has acknowledged.

**Aayush:** The current design assumes that messages need to be delivered in publish order to all subscribers. With the new requirement of delivering to any one random subscriber, we would need to track whether a message has been delivered to a subscriber or not in order for the message to be successfully deleted. And while delivering, instead of delivering across all subscription channels, we can do round robin where the last subscription index will be tracked so that we can get the subscriber index for sending the next message.

*(Round ends — 91 min.)*

---

## His Design

**Requirements he gathered:** 9 numbered requirements covering publish, subscribe, delivery, independent topic creation with unique names, exception on publish/subscribe to a missing topic, per-topic capacity set at creation, at-least-once delivery, publish-order delivery. Capacity-full behaviour (exception to the publisher) was added at the entity stage under probing, not in the requirements phase.

**Out of Scope:** none — never produced, never prompted.

**Entities & relationships:** first pass scrapped by him (`Message.status` couldn't survive multiple subscribers, `Subscriber` had no state). Final: `MessageBus(topics)` orchestrates; `Topic(name, capacity, messages, subscriptions)`; `Subscription(subscriber, topic, nextSequenceNumber)` as the per-subscriber cursor; `Message(id, payload, sequenceNumber)`; `Publisher(messageBus)`; `Subscriber(subscriptions)`.

**Class outlines:** see `pub_sub_message_bus_design.md` §3 — full state and signatures with return types and a named exception per failure mode, produced after prompting for return types.

**Core implementation:** see design sheet §4 — `addMessage`, `deliverPendingMessages`, `cleanupDeliveredMessages` in pseudo-code.

**Gaps / misplaced responsibilities:**
- `addMessage` declares `throws TopicCapacityExceededException` but its body contains **no capacity check**. He asserted under probing that "appending to the message list determines if publish succeeds depending on list size" — no such step exists in what he wrote.
- **Nothing ever calls `cleanupDeliveredMessages()`.** No method in the design invokes it, so `topic.messages` only grows. A fully-drained topic with a healthy acknowledged subscriber stays permanently at capacity and rejects every future publish.
- `deliverPendingMessages` delivers exactly one message per subscription per call despite the plural name, and its only trigger is a new publish — so a failed delivery is retried only if someone publishes again. "The message will be retried later" has no mechanism.
- `cleanupDeliveredMessages` skips cleanup entirely when there are no subscriptions; combined with the capacity rule, a topic with no subscribers fills and then rejects publishes permanently.
- `Subscription.getNextSequenceNumber()` exists purely so `Topic` can decide which message to send — Ask, not Tell. The cursor's owner never acts on it.
- `onMessage` appears only in the verbal trace and was added to `Subscriber` after a clarifying question; it was absent from the class outline.
- Retention rule makes the slowest subscriber the throughput limit for all publishers on a topic — never surfaced as a trade-off by either side during the round.

---

## Feedback Given

### Round conditions
- **Hints used: 0/2** — no ceiling from hints. The `lastMessageSent` cursor and the retention rule were found unaided, which is the hardest idea in this problem.
- **Requirements asked unprompted:** topic ownership/creation, publish-to-missing-topic, subscribe-to-missing-topic, capacity limit, delivery guarantee, payload type, name uniqueness. Seven questions covering lifecycle, error contract, bounds and semantics before writing a line — his strongest requirements phase in this round type.
- **Never asked:** sync vs async delivery; duplicate subscription by the same subscriber; fate of undelivered messages on unsubscribe; cross-topic ordering; whether the bus is used from multiple threads.
- **Out of Scope list: never produced.** Hard ceiling at 3.
- **Self-verification: No** — trace had to be requested, and was incomplete and wrong.

### Rubric
| Area | Score | Reason |
|---|---|---|
| Requirements & scoping | 3/5 | Seven sharp unprompted questions, all committed to writing — but no Out of Scope list, and capacity-full behaviour stayed unspecified until forced out at the entity stage. |
| Entity modelling | 4/5 | First list weak (a `status` field that can't survive two subscribers), but he scrapped it himself; `Subscription` as a first-class entity with its own cursor is exactly the modelling move this problem turns on. |
| Class design | 4/5 | Complete state and, after prompting, complete signatures with a named exception per failure mode. Docked for shipping v1 with no return types, and for `onMessage` living only in the verbal trace. |
| Responsibility placement | 2/5 | `Topic` asks `Subscription` for a number then decides what to send, delivers it, and advances the cursor. `Subscription` owns state it isn't allowed to act on. |
| Implementation & correctness | 1/5 | Two breaks asserted to be fine. |
| Simplicity & judgement | 5/5 | Zero patterns, zero ceremony, no speculative abstraction; replay handled by changing one constructor value rather than inventing a strategy. |
| Extensibility | 4/5 | Replay landed exactly on the seam. Competing-consumers correctly identified as a model change rather than pretending it was free. |
| Concurrency | 4/5 | Named the correctness category before reaching for a lock, identified all three pieces of shared state, chose per-topic/per-subscription granularity over one global lock, kept `onMessage` outside the lock, avoided deadlock by never holding both. Docked because cost came only after two pushes, in four words. |
| Communication | 3/5 | Clear reasoning and good self-correction, but answered one part of a two-part question three separate times, and returned a dependency instead of values when asked to trace. |

### The two correctness breaks

**1. `addMessage` declares an exception its body never throws.** Signature: `addMessage(payload): Message throws TopicCapacityExceededException`. Body: generate sequence number, create message, append, trigger delivery. No capacity check anywhere. When asked which step decides whether the publish succeeds, he pointed at a check that isn't written. Same pattern as the previous two rounds: declared contract and method body disagree, and the body is the one that runs.

**2. Nothing ever calls `cleanupDeliveredMessages()`** — the fatal one. `addMessage` triggers delivery; `deliverPendingMessages` advances cursors; no method anywhere invokes cleanup. `topic.messages` only grows.

Real trace for capacity 2 with A acknowledging everything:

| Step | Real state |
|---|---|
| publish m1 | `messages={m1}`, subscription advances to 2 after ack |
| publish m2 | `messages={m1,m2}`, subscription advances to 3 after ack |
| publish m3 | `messages={m1,m2,m3}` — appended, because there is no capacity check |

And had the check existed: m3 is rejected **forever**. A has acknowledged everything, the queue is logically empty, but cleanup is never called by anyone, so `messages` still holds `{m1,m2}`, the topic is permanently at capacity, and every future publish fails for the life of the process. A fully-drained topic with a healthy subscriber is bricked — not a subtle race, the main path. "It depends on whether cleanup ran" was the moment to ask *who runs cleanup*, and the answer was nobody.

Two smaller ones: `deliverPendingMessages` delivers exactly one message per subscription per call and is only triggered by a publish, so a failed delivery is never retried if publishing stops — at-least-once needs a clock, not a publish. And `cleanupDeliveredMessages` skipping cleanup when there are no subscriptions bricks an unsubscribed topic the same way.

### Responsibility placement
`Subscription` holds `nextSequenceNumber` and exposes `getNextSequenceNumber(): int`; the caller finds the matching message, delivers it, then calls back into `acknowledge()`. The object owns state whose only purpose is to let something else decide about it — Ask, not Tell, third round running. The Tell version: `Subscription` is told to deliver, knows its own cursor, asks the topic for the message at that position, invokes its own subscriber, advances itself on success. That also fixes the lock story for free — the subscription lock then protects state that only moves inside subscription methods.

### Pace report
Cut off during **class design**: at minute 40 he had just submitted the first class outline with no return types. No implementation, no concurrency question, neither extensibility follow-up would have been reached. Biggest sink: entities at 28.5 min against a 3-min reference, most of it a four-turn exchange where each answer covered one part of a two-part question. Requirements was the only near-pace phase — and his best. The pattern: fast when asking, slow when answering.

### Performance Rating: 2/5
**Would have been a 4 — capped at 2 by core logic with a flaw never caught.** The entity model is right, the sequence-number cursor is the correct mechanism, the lock design is senior-grade, and the simplicity is exemplary. But he shipped a publish path that can permanently brick a topic, asserted a capacity check that doesn't exist in his own code, and when handed the trace that would have exposed it, produced a dependency instead of values. The Out of Scope omission independently caps at 3; the uncaught flaw binds at 2.

### Senior-signal scorecard
| Signal | Read | Why |
|---|---|---|
| Scopes before designing | Mixed | Seven strong unprompted questions, but no Out of Scope list for the third round running, and capacity-full behaviour left as a gap. |
| State derived from requirements | Strong | Justified the sequence number as load-bearing unprompted; deleted `Message.status` the moment it failed a two-subscriber scenario. |
| Rules live with their state | Weak | The cursor lives in `Subscription` and is operated entirely from `Topic`. |
| Simplicity held under pressure | Strong | Four follow-ups, zero patterns invented, no defensive abstraction. |
| Verifies own logic | Weak | No trace offered; the requested trace returned a conditional and missed a main-path break. |
| Extends without rewriting | Strong | Replay landed on the subscription-creation seam in one sentence; competing-consumers correctly scoped as a model change. |

**Overall: mid-level, trending senior on modelling and judgement, held back by verification. No-hire on this round** — the design would pass, the code would not.

### What a senior strong-hire would have done
- **Asked whether the bus is multi-threaded**, in requirements. It's an in-process message bus — concurrency is the point, not a follow-up. That one question changes where the locks live instead of bolting them on.
- **Written the Out of Scope list**: persistence, cross-topic ordering, dead-letter after N failures, wildcard subscriptions, backpressure on slow subscribers. Thirty seconds, third round skipped.
- **Named who calls cleanup at the moment cleanup was invented.** The natural answer: `acknowledge` triggers it — cleanup is a consequence of a cursor advancing, so it belongs on that path, not floating as an orphan.
- **Put the capacity check as line one of `addMessage`,** and noticed that "reject when full" plus "only clean up after everyone acks" means **one slow subscriber blocks every publisher on that topic**. That's the real design tension and neither side said it out loud; a senior surfaces it and either accepts it or offers drop-oldest / evict-the-laggard.
- **Traced with values.** Capacity 2 and three publishes was chosen precisely to force a statement about the queue after m3.

### One drill
Before writing any method body, write the name of the caller next to every method in the class outline. Any method with no caller is either dead or a bug — `cleanupDeliveredMessages` was the second kind. Ninety seconds, and it catches this round's fatal break.
