# In-Process Pub/Sub Message Bus — Design Sheet

## 1. Requirements

1. Publishers can push messages to a topic
2. Subscribers can subscribe to a topic
3. Subscribers of a topic are delivered messages which have been pushed to the topic
4. Topic creation independent of publish and subscribe. Topic names uniquely identify topics
5. Publishing to a non-existent topic throws exception
6. Subscribing to a non-existent topic throws exception
7. Each topic is bounded by a capacity set while creating topic
8. At-least-once delivery guarantee for messages
9. Messages must be delivered in publish order

## Out of Scope

_(none stated)_

## 2. Entities & Relationships

*(first list, superseded — "does not cover many entities")*
```
MessageBus (topics: Topic[], topicSubscribersMap)
Subscriber
Publisher (messageBus: MessageBus)
Message (status: DELIVERED | NOT_DELIVERED)
Topic (name, capacity, messages: Message[])
```

*(revised list)*
```
MessageBus   (topics: Topic[])
Subscriber   (subscriptions: Subscription[])
Publisher    (messageBus: MessageBus)
Subscription (subscriber, topic, lastMessageSent)
Message      (payload, id)
Topic        (name, capacity, messages: Message[], subscriptions: Subscription[])
```

**Retention rule:** a message can be removed from the topic queue only once it has been sent to
all subscriptions — i.e. `lastMessageSent` of every subscription >= the message to be deleted.

**Capacity rule:** if the topic is at capacity and the oldest message cannot yet be removed, the
publish is rejected with an exception surfaced to the publisher (topic capacity reached).

## 3. Class Design

```
class MessageBus
    - topics: Map<TopicName, Topic>

    + createTopic(topicName, capacity): void
        throws TopicAlreadyExistsException, InvalidCapacityException
    + publish(topicName, payload): void
        throws TopicNotFoundException, TopicCapacityExceededException
    + subscribe(topicName, subscriber): Subscription
        throws TopicNotFoundException
    + unsubscribe(topicName, subscriber): void
        throws TopicNotFoundException, SubscriptionNotFoundException


class Publisher
    - messageBus: MessageBus

    + publish(topicName, payload): void
        throws TopicNotFoundException, TopicCapacityExceededException


class Subscriber
    - subscriptions: List<Subscription>

    + addSubscription(subscription): void
    + removeSubscription(subscription): void
        throws SubscriptionNotFoundException
    + onMessage(message)          // added later: executes subscriber code on delivery


class Subscription
    - subscriber: Subscriber
    - topic: Topic
    - nextSequenceNumber: int

    + getNextSequenceNumber(): int
    + acknowledge(sequenceNumber): void
        throws InvalidSequenceNumberException


class Topic
    - name: String
    - capacity: int
    - messages: Queue<Message>
    - subscriptions: List<Subscription>
    - nextSequenceNumber: int

    + addSubscription(subscription): void
    + removeSubscription(subscription): void
        throws SubscriptionNotFoundException
    + addMessage(payload): Message
        throws TopicCapacityExceededException
    + deliverPendingMessages(): void
    + cleanupDeliveredMessages(): void


class Message
    - id: String
    - payload: String
    - sequenceNumber: int
```

**On sequence numbers (his words):** the sequence number is load-bearing — it determines which
message must be sent next across a subscription channel. It is a monotonically increasing integer
assigned by the Topic.

**Delivery flow (his trace):** `bus.subscribe` looks up the topic, creates a `Subscription`
(topic, subscriber, nextSequenceNumber = 1) and adds it to both the topic and the subscriber.
`publish` looks up the topic and delegates to `Topic.addMessage(payload)`, which creates a
`Message` with the next sequence number and queues it. The topic then asynchronously triggers
`deliverPendingMessages()`, which finds the message matching each subscription's
`nextSequenceNumber` and invokes `subscription.subscriber.onMessage(message)`. On successful
processing the subscriber acknowledges through the subscription, advancing `nextSequenceNumber`.
On failure the number does not advance, so the message is redelivered — at-least-once, and the
sequence number preserves order.

`getNextSequenceNumber()` is called by the async delivery mechanism to decide which message to
deliver across a subscription channel.

## 4. Core Implementation (pseudo-code)

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

**Trace (capacity 2, subscriber A, publish m1/m2/m3) — as he stated it:**
`messages -> {m1, m2}`; `topic.nextSequenceNumber -> 2`; `subscription.nextSequenceNumber -> 1`
when m1 not pushed, 2 when m1 pushed, 3 when m2 pushed. The m3 publish succeeding "depends
entirely on whether any of the previous messages have been cleaned up or not"; the appending to
the message list determines if publish succeeds, depending on list size.

## 5. Follow-ups (as answered)

**Concurrency** — two publishers racing while delivery and cleanup run:
contention on the messages list; two concurrent publishes could be allotted the same sequence
number, breaking publish order; and if one of them is delivered, cleanup could remove both while
one was never delivered — "a correctness issue introduced due to concurrency."
Fix: a **topic lock** protecting `topic.nextSequenceNumber` and `topic.messages` in `addMessage`,
and a **subscription lock** protecting `subscription.nextSequenceNumber` in the delivery path so
only one delivery attempt per subscription channel advances the cursor.
Cost: "the topic lock costs publish throughput at runtime." `onMessage` must run **outside** the
lock — the topic lock is only held to fetch the message to be delivered.
Lock ordering: cleanup takes the subscription lock to compute the minimum sequence number,
releases it, then acquires the topic lock to delete — never holding both at once.

**Extensibility 1 — replay of retained history for some subscribers:**
the `nextSequenceNumber` set at subscription-creation time should be decided based on the topic's
message list at that moment. The caller expresses which behaviour it wants via an argument to
`MessageBus.subscribe()`.

**Extensibility 2 — competing consumers (exactly one subscriber per message):**
the current design assumes delivery in publish order to *all* subscribers. With delivery to one
subscriber, we would need to track whether a message has been delivered to a subscriber at all in
order to delete it safely; and instead of delivering across all subscription channels, do round
robin, tracking the last subscription index to pick the next subscriber.

---
---

# Optimal Reference (what a senior strong-hire would design)

Everything below is the teaching the round withheld. It includes what was missed. His design above
is untouched.

## 1. Requirements + Out of Scope

**Functional**
1. `createTopic(name, capacity)` — explicit; names are unique; duplicate is an error.
2. `publish(topicName, payload)` — error if the topic does not exist.
3. `subscribe(topicName, handler, startFrom)` — error if the topic does not exist; returns a
   handle the caller can later use to unsubscribe.
4. Each subscriber receives every message published to the topic after its start position,
   **in publish order**, **at least once**.
5. Delivery is **asynchronous** — `publish` returns without waiting for any subscriber.
6. A topic retains at most `capacity` messages; a message is retained until every subscription
   has acknowledged it.
7. `unsubscribe` removes a subscription and releases its hold on retained messages.

**Questions he never asked, and what each would have changed**
- *Is the bus used from multiple threads?* — It's an in-process bus, so yes. Asking this in
  phase 1 makes the lock placement part of the class design instead of a retrofit.
- *Can the same subscriber subscribe to a topic twice?* — Decides whether `Subscriber → Subscription`
  is 1:1 or 1:N per topic, and whether `unsubscribe(topic, subscriber)` is even unambiguous.
  (It isn't, if duplicates are allowed — which is why the optimal API returns a handle.)
- *What happens to undelivered messages when a subscriber unsubscribes?* — They become eligible
  for cleanup immediately; without this, an unsubscribed laggard pins the queue forever.
- *What happens when a subscriber's handler keeps failing?* — Retry forever, or dead-letter after
  N attempts? "At-least-once" without a give-up rule is an infinite redelivery loop.
- *Must ordering hold across topics?* — No. Saying so out loud is what lets each topic own its own
  lock instead of needing a global one.

**Out of Scope** *(the list that was never written)*
- Persistence / durability across process restart — the bus is in-memory.
- Cross-topic ordering guarantees.
- Wildcard or pattern-matched topic subscriptions.
- Exactly-once delivery and subscriber-side de-duplication.
- Authentication / authorisation on publish or subscribe.
- Metrics, tracing, and admin APIs.

## 2. Entities & relationships

```
MessageBus  ── owns ──▶  Topic (by name)          ← the orchestrator / facade
Topic       ── owns ──▶  Message ring buffer
Topic       ── owns ──▶  Subscription[]
Subscription ── holds ─▶ MessageHandler (the subscriber's callback)
Subscription ── holds ─▶ its own cursor, and is the ONLY thing that moves it
DeliveryScheduler ── drives ──▶ Subscription.deliverNext()   ← the missing piece
```

**Orchestrator: `MessageBus`.** It owns the topic registry and nothing else — lookup, lifecycle,
and error translation. Every rule about messages lives in `Topic`; every rule about *one
subscriber's progress* lives in `Subscription`.

**The entity his design lacks: `DeliveryScheduler`** (a worker pool / executor). Without it,
delivery and retry have no engine, which is why `cleanupDeliveredMessages` ended up with no caller
and why a failed delivery was only retried on the next publish.

## 3. Class outlines

```
class MessageBus
    - topics: Map<String, Topic>
    - scheduler: DeliveryScheduler
    - lock: ReadWriteLock                  // guards the registry only

    + createTopic(name: String, capacity: int): Topic
        throws TopicAlreadyExistsException, InvalidCapacityException
    + publish(topicName: String, payload: Object): void
        throws TopicNotFoundException, TopicFullException
    + subscribe(topicName: String, handler: MessageHandler,
                startFrom: StartPosition): SubscriptionHandle
        throws TopicNotFoundException
    + unsubscribe(handle: SubscriptionHandle): void
        throws SubscriptionNotFoundException
    + shutdown(): void


enum StartPosition { LATEST, EARLIEST_RETAINED }


interface MessageHandler                   // what a "Subscriber" actually is
    + onMessage(message: Message): void     // throwing = negative ack = redeliver


class Topic
    - name: String
    - capacity: int
    - messages: Deque<Message>             // ordered by sequence, oldest first
    - subscriptions: List<Subscription>
    - nextSequence: long                   // monotonic, assigned here
    - lock: Object                         // guards messages + nextSequence + subscriptions

    + append(payload: Object): Message
        throws TopicFullException          // capacity check is the FIRST thing it does
    + messageAt(sequence: long): Optional<Message>
    + addSubscription(s: Subscription): void
    + removeSubscription(s: Subscription): void
    + oldestRetainedSequence(): long       // for StartPosition.EARLIEST_RETAINED
    + nextSequenceToAssign(): long         // for StartPosition.LATEST
    - pruneAcknowledged(): void            // PRIVATE — called on every cursor advance


class Subscription
    - id: String
    - topic: Topic
    - handler: MessageHandler
    - cursor: long                         // next sequence this subscriber must receive
    - failureCount: int
    - inFlight: boolean                    // one delivery at a time per subscription
    - lock: Object                         // guards cursor + inFlight + failureCount

    + deliverNext(): DeliveryOutcome       // TELL, not ask — no getCursor()
    + hasPending(): boolean
    + acknowledgedThrough(): long          // read-only, used by pruning
    + close(): void


class Message
    - id: String
    - sequence: long
    - payload: Object                      // opaque; the bus never inspects it
    - publishedAt: Instant


class DeliveryScheduler
    - executor: ThreadPool
    - retryTicker: ScheduledExecutor       // the clock that makes at-least-once real

    + schedule(subscription: Subscription): void
    + scheduleRetry(subscription: Subscription, delay: Duration): void
    + shutdown(): void


enum DeliveryOutcome { DELIVERED, NOTHING_PENDING, FAILED_RETRY, FAILED_DEAD_LETTER }
```

**Note what is absent:** no `Publisher` class. A publisher is just any caller holding a reference
to the bus — modelling it as a class adds a type that owns no state and enforces no rule. Likewise
`Subscriber` collapses to a `MessageHandler` interface: the "subscriber" in this domain *is* the
callback plus its cursor, and the cursor already has a home in `Subscription`.

## 4. Core implementations (pseudo-code)

```
Topic.append(payload):
    lock(this.lock):
        pruneAcknowledged()                          # reclaim before rejecting
        if messages.size >= capacity:
            throw TopicFullException(name, capacity) # THE CHECK — before any mutation
        seq = nextSequence++
        msg = new Message(uuid(), seq, payload, now())
        messages.addLast(msg)
        subs = copyOf(subscriptions)                 # snapshot inside the lock
    # outside the lock — never hold a lock while handing work to the scheduler
    for s in subs:
        scheduler.schedule(s)
    return msg


Subscription.deliverNext():
    lock(this.lock):
        if inFlight:            return NOTHING_PENDING   # one at a time, preserves order
        msg = topic.messageAt(cursor)
        if msg is absent:       return NOTHING_PENDING
        inFlight = true
    # handler runs OUTSIDE the lock: a slow subscriber must not block publishers,
    # cleanup, or its own topic's other subscribers
    try:
        handler.onMessage(msg)
        lock(this.lock):
            cursor  = msg.sequence + 1
            failureCount = 0
            inFlight = false
        topic.onCursorAdvanced()        # <-- THE CALLER cleanup was missing
        return DELIVERED
    catch (Exception e):
        lock(this.lock):
            failureCount++
            inFlight = false
            attempts = failureCount
        if attempts >= MAX_ATTEMPTS:
            deadLetter(msg, e)
            lock(this.lock): cursor = msg.sequence + 1   # skip the poison message
            topic.onCursorAdvanced()
            return FAILED_DEAD_LETTER
        scheduler.scheduleRetry(this, backoff(attempts)) # retry has a CLOCK, not a publish
        return FAILED_RETRY


Topic.onCursorAdvanced():
    lock(this.lock):
        pruneAcknowledged()

Topic.pruneAcknowledged():          # caller already holds this.lock
    if subscriptions.isEmpty():
        # a topic with no subscribers must still drain, or it bricks at capacity
        messages.clear()
        return
    minAcked = min(s.acknowledgedThrough() for s in subscriptions)
    while messages.isNotEmpty() and messages.peekFirst().sequence < minAcked:
        messages.removeFirst()
```

**Trace — capacity 2, one subscriber A acknowledging each message, publish m1, m2, m3:**

| Event | `messages` | `topic.nextSequence` | `A.cursor` |
|---|---|---|---|
| start | `{}` | 1 | 1 |
| publish m1 | `{m1}` | 2 | 1 |
| A acks m1 → prune (minAcked=2) | `{}` | 2 | 2 |
| publish m2 | `{m2}` | 3 | 2 |
| A acks m2 → prune (minAcked=3) | `{}` | 3 | 3 |
| publish m3 | `{m3}` | 4 | 3 |
| A acks m3 → prune | `{}` | 4 | 4 |

Three publishes, capacity 2, **nothing rejected** — because pruning is driven by the cursor
advancing. Under his design the queue is `{m1,m2,m3}` (no capacity check) or permanently `{m1,m2}`
with every subsequent publish failing forever (check added, cleanup never invoked).

**Edge cases the implementation must handle**
- Publish to a full topic where the oldest message genuinely can't be pruned (a real laggard) →
  `TopicFullException`. That is backpressure, and it is *correct* — but see the trade-off below.
- Subscriber throws on every attempt → dead-letter after `MAX_ATTEMPTS`, then skip, so one poison
  message cannot pin the queue forever.
- Unsubscribe while a delivery is in flight → `close()` marks the subscription dead; the in-flight
  ack is discarded, and the subscription is removed before the next prune so it stops pinning.
- Last subscriber unsubscribes → `pruneAcknowledged` clears the queue rather than freezing it.
- Subscribe with `EARLIEST_RETAINED` on an empty topic → cursor = `nextSequenceToAssign()`.

## 5. Design decisions vs. named alternatives

| Decision | Alternative | What it gives up |
|---|---|---|
| Per-subscription cursor over a shared queue | A separate copy of the queue per subscriber | Cursors cost O(1) memory per subscriber; per-subscriber queues cost O(messages × subscribers) and make ordering-across-subscribers untrackable. Given up: subscribers cannot be at wildly different positions beyond `capacity`. |
| Pruning driven by cursor advance | A background sweeper thread | The sweeper is simpler to write but adds a second concurrent mutator of `messages` and leaves a window where a full topic rejects publishes it could have accepted. Given up: prune cost is paid on the ack path. |
| `MessageHandler` interface, no `Subscriber`/`Publisher` classes | Concrete `Subscriber` and `Publisher` classes | Two fewer types that hold no state and enforce no rule. Given up: nothing — a publisher that needs identity can hold a bus reference plus its own fields. |
| Per-topic lock + per-subscription lock | One global bus lock | Publishers to different topics never contend. Given up: two lock types means an ordering discipline (never hold both; the handler runs outside both). |
| `throws TopicFullException` on a full topic | Drop-oldest, or block the publisher | Explicit backpressure the publisher can react to. Given up: a single slow subscriber can fail publishes for everyone — the real tension in this problem, and it should be *stated* even if accepted. |
| **No design pattern** | Strategy for delivery mode, Observer for subscribers, Factory for topics | `StartPosition` is an enum read once at construction, and "Observer" is just what pub/sub *is* — naming it adds a word, not a seam. Given up: nothing. A Strategy earns its place only when competing-consumer mode lands (see below), and not before. |

## 6. Extensibility — where each follow-up lands

- **Replay from retained history** → `StartPosition` at subscribe time sets the initial cursor to
  `topic.oldestRetainedSequence()` instead of `topic.nextSequenceToAssign()`. One value at
  construction. No class changes. *(He found this seam.)*
- **Competing consumers** → this is the one follow-up that genuinely does not fit a per-subscription
  cursor, because "delivered to exactly one of N" is not expressible as N independent cursors. The
  seam is `Subscription` itself: introduce `ConsumerGroup` as a *sibling* of `Subscription` in the
  topic's subscription list — it holds one shared cursor plus a round-robin index and a per-message
  in-flight/ack map. `Topic.pruneAcknowledged` already asks each entry for `acknowledgedThrough()`,
  so it needs no change at all. *This* is where a Strategy (or a common `DeliveryTarget` interface)
  finally earns its keep, and not one moment earlier.
- **Dead-letter topic** → `deadLetter(msg, e)` already exists as a seam; point it at another `Topic`.
- **Per-subscriber filtering / predicates** → a decorator around `MessageHandler`; the bus never
  learns about it.

## 7. Concurrency

**Categories, all three present:**
- **Correctness** — `nextSequence` assignment and the `messages` deque are read-modify-write under
  concurrent publishes; `cursor` is check-then-act under concurrent deliveries.
- **Coordination** — publishers hand work to delivery threads; `DeliveryScheduler` is the handoff.
- **Scarcity** — bounded `capacity` per topic, and a bounded delivery thread pool.

**Primitives and where the synchronization actually lives:**
- `Topic.lock` guards `nextSequence`, `messages`, and `subscriptions`. Held for the append and the
  prune only — never across a handler call.
- `Subscription.lock` guards `cursor`, `inFlight`, `failureCount`. `inFlight` is what makes
  concurrent delivery attempts on one subscription impossible, which is what preserves per-subscriber
  ordering under a multi-threaded scheduler.
- **Lock ordering: never hold both.** `append` takes the topic lock and releases it before scheduling.
  `deliverNext` takes the subscription lock, releases it, runs the handler, re-takes it, releases it,
  *then* calls into the topic. No path holds a subscription lock while acquiring a topic lock, so
  there is no cycle and no deadlock. *(He arrived at exactly this discipline for cleanup.)*
- The handler runs outside every lock. A subscriber that blocks for a minute delays only its own
  subscription.

**Costs, stated plainly:**
- The topic lock serialises all publishers to that topic. Throughput ceiling is one append at a
  time per topic; different topics are fully parallel. If one topic becomes hot, the next step is
  striping the deque or moving to a lock-free ring buffer with a CAS'd sequence counter — not a
  bigger lock.
- `inFlight` caps each subscription at one message at a time. That is the price of ordered
  at-least-once: a slow subscriber cannot be parallelised without giving up order.
- Pruning on the ack path puts an O(subscriptions) `min` computation on every acknowledgement.
  Fine at tens of subscribers; at thousands, keep an incrementally-maintained min instead.
- The retry ticker means a permanently-failing subscriber consumes scheduler slots forever until it
  dead-letters — which is why `MAX_ATTEMPTS` is not optional.


_(none stated)_
