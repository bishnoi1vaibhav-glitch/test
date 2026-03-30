# Problem Statement & Root Cause Analysis
## Kafka In-Flight Message Loss on Pod Shutdown

**Component:** `axis-kafka-starter` — `KafkaTopicConsumer`

---

## 1. Executive Summary

Services built on the Maximus platform consume Kafka messages via `KafkaTopicConsumer<T>`, a shared base class in `axis-kafka-starter`. On pod shutdown (SIGTERM), Spring Boot destroys the MongoDB connection while Kafka consumers are still mid-processing. The reactive pipeline loses its database session, the business operation silently fails, but the Kafka offset has already been committed — leaving the application record stuck in an intermediate state with no automatic recovery.

This is not a transient error. The message is permanently lost from the queue. The only recovery path is a manual MongoDB remediation script run by product teams.

**Monthly impact: approximately 261 + 4846 application records across 5 products are stuck due to this cause.**

---

## 2. Background

### 2.1 KafkaTopicConsumer architecture

All Kafka consumption in the platform is centralised in `KafkaTopicConsumer<T>` in `axis-kafka-starter`. Each concrete consumer:

- Creates a `KafkaReceiver` subscription on `ApplicationReadyEvent`
- Processes each record through a `flatMapSequential` reactive pipeline
- Commits the Kafka offset via `receiverOffset.commit()` at the end of the chain
- Interacts with MongoDB via `ReactiveMongoTemplate` within the processing chain

The `worker` field holds the `Disposable` returned by `.subscribe()` on the receiver chain. This is a plain object reference — it is not registered anywhere in Spring's lifecycle.

### 2.2 Spring graceful shutdown scope

Spring Boot's `server.shutdown: graceful` drains in-flight HTTP requests before destroying beans. This works because Netty/Reactor Netty registers itself as a `SmartLifecycle` bean and participates in the ordered shutdown sequence.

`KafkaTopicConsumer` workers are not `SmartLifecycle` beans. They receive no shutdown signal from Spring. Spring proceeds to destroy all beans — including `MongoClient` — without waiting for active Kafka processing chains to complete.

---

## 3. Failure Sequence

The following sequence occurs on every pod termination where a Kafka message is in-flight:

```
T+0s    SIGTERM received by pod
        │
        ├── Spring: Netty stops accepting HTTP requests
        │   Spring: waits for active HTTP handlers to complete    ← works correctly
        │
T+~2s   Spring: begins OrderedBean destruction (phase MAX_VALUE)
        │
        ├── MongoClient bean destroyed
        │   MongoDB ClientSession pool transitions to CLOSED
        │
        │   ← KafkaTopicConsumer worker: still executing
        │       flatMapSequential pipeline in progress
        │       pending: ReactiveMongoTemplate.find() / save()
        │
T+~2s   ReactiveMongoTemplate attempts MongoDB operation
        │
        └── ClientSessionBinding.getReadConnectionSource()
                ↓
            BaseCluster.selectServerAsync()
                ↓
            Assertions.isTrue(state == OPEN)  →  FALSE
                ↓
            IllegalStateException: state should be: server session pool is open
                ↓
            MongoExceptionTranslator → ClientSessionException
                ↓
            Propagates up reactive chain
                ↓
            No onError handler at subscribe boundary
                ↓
            reactor.core.publisher.Operators → onErrorDropped
            ← ERROR logged, exception silently swallowed
        │
        Kafka offset: ALREADY COMMITTED at record receipt
        Application record: STUCK in intermediate state
        Message: NOT redelivered (offset committed)
```

---

## 4. Observed Error Signature

This failure produces two observable error patterns in logs:

### Pattern A — `onErrorDropped` (primary, silent)

```json
{
  "logger": "reactor.core.publisher.Operators",
  "message": "Operator called default onErrorDropped",
  "level": "ERROR",
  "stack": "reactor.core.Exceptions$ErrorCallbackNotImplemented: org.springframework.data.mongodb.ClientSessionException: state should be: server session pool is open\nCaused by: org.springframework.data.mongodb.ClientSessionException: state should be: server session pool is open\n\tat org.springframework.data.mongodb.core.MongoExceptionTranslator.doTranslateException(MongoExceptionTranslator.java:159)\n\tat org.springframework.data.mongodb.core.MongoExceptionTranslator.translateExceptionIfPossible(MongoExceptionTranslator.java:74)\n\tat org.springframework.data.mongodb.core.ReactiveMongoTemplate.potentiallyConvertRuntimeException(ReactiveMongoTemplate.java:2783)\n\tat org.springframework.data.mongodb.core.ReactiveMongoTemplate.lambda$translateException$100(ReactiveMongoTemplate.java:2766)\n\tat reactor.core.publisher.MonoFlatMap$FlatMapMain.secondError(MonoFlatMap.java:241)\n\tat reactor.core.publisher.MonoFlatMap$FlatMapInner.onError(MonoFlatMap.java:315)"
}
```

This is the critical pattern — the error is swallowed by the Reactor default `onErrorDropped` handler. No retry is triggered. No alert fires. The message is silently lost.

### Pattern B — `MLP-814` (secondary, logged by orchestrator)

```json
{
  "errorCode": "MLP-814",
  "message": "Error while processing Step Completion Event",
  "stepId": "InitiateCardFund",
  "stack": "org.springframework.data.mongodb.ClientSessionException: state should be: open\n\tat BaseCluster.selectServerAsync\n\tat ClientSessionBinding.getReadConnectionSource\n\tat FindOperation.executeAsync"
}
```

This pattern is caught by the orchestrator's error handler and logged with a structured error code, but the outcome is the same — the step is not completed.

Both errors originate from the same root cause. Pattern A is more dangerous because it is structurally silent.

---

## 5. Root Cause Analysis

### 5.1: Kafka workers not registered in Spring lifecycle

`KafkaTopicConsumer.run()` calls `.subscribe()` and stores the result as a private `Disposable worker`. This `Disposable` has no connection to Spring's `ApplicationContext` lifecycle. When Spring shuts down, it has no knowledge of active Kafka subscriber chains.

```kotlin
// Current implementation — worker is invisible to Spring lifecycle
worker = getKafkaReceiver(sslToggle).receive()
    .flatMapSequential { ... }
    .flatMap { it.commit() }
    .subscribe()   // ← Disposable stored, but not registered anywhere
```

### 5.2: No `onError` boundary at the subscribe level

The `flatMapSequential` pipeline has per-record `onErrorResume` handlers, but these are designed for business logic errors (retryable failures, DLQ routing). The `subscribe()` call at the terminal end of the chain has no `onError` consumer. When an unexpected error (like `ClientSessionException`) bypasses the per-record handlers, it reaches the subscribe boundary and is swallowed by Reactor's default `onErrorDropped`.

### 5.3: MongoClient destruction precedes Kafka drain

Spring's default bean destruction phase is `Integer.MAX_VALUE`. Both `MongoClient` and `KafkaTopicConsumer` workers effectively compete at the same phase, with no guaranteed ordering. In practice, MongoDB beans are destroyed first, before active Kafka chains complete.

---

## 6. Impact Assessment

### 6.1 Monthly stuck applications (shutdown-caused)

| Product | Applications stuck / month |
|---|---|
| Forex | 44 |
| Credit Card | 18 + 23(gpay) |
| Auto Loan | 67 |
| Personal Loan | 109 |
| dr-service* | 4,846 |
| **Total** | **261 + 4,846** |

### 6.2 Operational cost

Each stuck application requires a product-team-operated MongoDB remediation script to identify and advance the workflow state. This is a manual operational burden running monthly across multiple teams. The scripts treat the symptom — the platform defect is the cause.

### 6.3 Risk profile

| Risk | Severity |
|---|---|
| Message permanently lost — no redelivery | Critical |
| Error is structurally silent (`onErrorDropped`) — no alerting | High |
| Affects every pod termination event (deploy, scale-down, restart) | High |
| Worsens at higher deployment frequency | High |
| Product teams have no visibility without log analysis | Medium |

---

## 7. Conditions That Trigger This Failure

| Trigger | Frequency |
|---|---|
| Rolling deployment | Every deployment across all services |
| HPA scale-down after load spike | Multiple times per day per service |
| Manual pod restart / node eviction | On demand / infra events |
| Kubernetes liveness probe failure restart | On demand |

Any event that sends SIGTERM to a pod running an active Kafka consumer is a potential trigger. Given deployment frequency across the platform, this occurs many times per day.

---

## 8. Why Existing Mechanisms Do Not Prevent This

| Mechanism | Why it does not help |
|---|---|
| `server.shutdown: graceful` | Covers HTTP only — Kafka not in scope |
| Per-record `onErrorResume` in consumer | Handles business errors, not infrastructure teardown errors |
| Kafka retry / DLQ | Not reached — offset already committed before failure |

---

## 9. References
Kibana: [https://kibana.maximus.axisb.com/goto/b3671c4bd53f490bd9355abb482657c2]
- `KafkaTopicConsumer.kt` — `axis-kafka-starter`
- `DynamicTopicNameResolver.kt` — `axis-kafka-starter`
- `KafkaTopicProducer.kt` — `axis-kafka-starter`
- Spring `SmartLifecycle` — [https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/SmartLifecycle.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/SmartLifecycle.html)
- Reactor `onErrorDropped` — [https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Operators.html](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Operators.html)
