---
title: "카프카 톺아보기 — Consumer Group, Offset, 파티션의 진짜 동작"
date: 2026-05-02 10:00:00 +0900
categories: [Kafka]
tags: [kafka, consumer-group, offset, partition, reactor, kotlin-coroutines]
toc: true
---

---

## 들어가며

이 글에서는 Consumer Group·Offset·파티션의 **실제 동작 방식**을 코드와 다이어그램으로 정리한다. 예제 코드는 Reactor Kafka + Kotlin Coroutines 기반의 상품 인덱싱 컨슈머를 사용한다.

---

## Consumer Group — 파티션 할당과 Fail-Over

### 기본 구조

카프카의 Consumer Group은 **고가용성(HA)을 위한 Fail-Over 메커니즘**이다.

```
┌─────────────────────────────────────────────────────────┐
│          Consumer Group: "product-consumer-group"        │
│                                                         │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│   │Consumer A│  │Consumer B│  │Consumer C│             │
│   │ (active) │  │(standby) │  │(standby) │             │
│   └────┬─────┘  └──────────┘  └──────────┘             │
│        │                                                │
└────────┼────────────────────────────────────────────────┘
         │
    ┌────▼─────┐
    │ Topic    │
    │ [P0][P1] │
    └──────────┘
```

핵심 규칙:

- **하나의 파티션**은 그룹 내 **하나의 컨슈머**에게만 할당된다
- 그룹 내 나머지 컨슈머는 **스탠바이 상태**로 대기한다
- 활성 컨슈머에 장애가 발생하면 스탠바이 컨슈머가 파티션을 이어받는다 (**리밸런싱**)

> 📌 Apache Kafka 공식 문서:
> *"Each partition is consumed by exactly one consumer within each subscribing consumer group."*
> — [Kafka Documentation: Consumer Groups](https://kafka.apache.org/documentation/#intro_consumers)

### 컨슈머 수와 파티션 수의 관계

파티션 수보다 컨슈머가 많으면, 초과된 컨슈머는 놀게 된다:

```
  Topic: 2 partitions, Consumer Group: 3 consumers
  ─────────────────────────────────────────────────

  Partition 0  ──▶  Consumer A (active)
  Partition 1  ──▶  Consumer B (active)
                    Consumer C (idle — 할당받을 파티션 없음)
```

반대로 컨슈머보다 파티션이 많으면, 하나의 컨슈머가 여러 파티션을 담당한다:

```
  Topic: 4 partitions, Consumer Group: 2 consumers
  ─────────────────────────────────────────────────

  Partition 0  ─┐
  Partition 1  ─┤──▶  Consumer A
               │
  Partition 2  ─┐
  Partition 3  ─┤──▶  Consumer B
```

### group.id는 논리적 소비 단위

`group.id`가 같으면 카프카는 하나의 Consumer Group으로 인식한다. **서로 다른 비즈니스 로직을 처리하는 컨슈머는 반드시 다른 `group.id`를 사용해야 한다.**

```kotlin
// 상품 인덱싱 컨슈머
@KafkaListener(
    topics = ["product-updated"],
    groupId = "product-indexing-group"
)
fun consumeProductUpdate(record: ConsumerRecord<String, String>) {
    productIndexService.updateIndex(record.value())
}

// 재고 집계 컨슈머 — 반드시 다른 group.id
@KafkaListener(
    topics = ["stock-updated"],
    groupId = "stock-summary-group"
)
fun consumeStockUpdate(record: ConsumerRecord<String, String>) {
    stockSummaryService.updateSummary(record.value())
}
```

같은 `group.id`를 사용하면 한쪽 컨슈머가 스탠바이로 밀려나 아무 일도 하지 못하게 된다.

---

## Offset — 가장 흔한 오해

### 오해: ack를 보내야 다음 메시지를 읽는다

많은 개발자가 카프카의 `acknowledge()`를 RabbitMQ의 ack처럼 생각한다. **하지만 카프카의 offset commit은 메시지 흐름을 제어하지 않는다.**

### 실제 동작: poll은 ack와 무관하게 계속 진행된다

```
           Offset의 실제 용도
           ─────────────────

  ┌─────────────────────────────────────────────────────────┐
  │                     Kafka Broker                        │
  │                                                         │
  │  Partition 0:  [msg0][msg1][msg2][msg3][msg4][msg5]...  │
  │                              ▲                          │
  │                    committed offset = 2                 │
  │                                                         │
  └──────────────────────────────┼──────────────────────────┘
                                 │
            ┌────────────────────┘
            │  재시작/리밸런싱 시
            │  여기부터 다시 읽기 시작
            ▼
  ┌──────────────────────────────────────────────────────┐
  │                    Consumer                          │
  │                                                      │
  │  poll() → [msg2, msg3, msg4, msg5, msg6, ...]       │
  │           ───────────────────────────────────        │
  │           ack와 무관하게 계속 읽어들임                  │
  │                                                      │
  │  acknowledge(msg4)  → committed offset = 4로 갱신    │
  │  → 다음 재시작 시 msg4부터 시작                        │
  └──────────────────────────────────────────────────────┘
```

> 📌 Apache Kafka 공식 문서:
> *"The position of the consumer gives the offset of the next record that will be given out. It will be one larger than the highest offset the consumer has seen in that partition. It automatically advances every time the consumer receives messages in a call to poll."*
> — [Kafka Documentation: Consumer Position](https://kafka.apache.org/documentation/#design_consumerposition)

**Offset의 용도는 딱 하나**: 장애 복구·재시작·리밸런싱 시 **읽기 시작 위치를 기억**하는 것이다.

### 이 오해가 만드는 문제 — 세마포어 + launch 패턴

offset의 동작을 오해하면 다음과 같은 안티패턴을 작성하게 된다:

```kotlin
fun startConsume(topic: String, groupId: String, permits: Int) {
    val semaphore = Semaphore(permits)

    kafkaReceiver(topic, groupId)
        .receive()
        .subscribe { record ->
            CoroutineScope(Dispatchers.Default).launch {
                semaphore.withPermit {
                    processRecord(record)
                    record.receiverOffset().acknowledge()
                }
            }
        }
}
```

"세마포어로 동시 처리 수를 제한하고, ack를 보내야 다음 메시지를 읽으니까 안전하다"고 생각하지만, 실제로는:

```
  receive()가 메시지를 계속 전달 (ack와 무관)
         │
         ▼
  subscribe { ... }  ←── 메시지마다 launch 코루틴 생성
         │
         ▼
  ┌───────────────────────────────────────┐
  │  launch {                             │
  │    semaphore.withPermit { process() } │  ← permit 대기
  │  }                                    │
  │  launch {                             │
  │    semaphore.withPermit { process() } │  ← permit 대기
  │  }                                    │
  │  launch {                             │
  │    semaphore.withPermit { process() } │  ← permit 대기
  │  }                                    │
  │  ... (대기 코루틴이 힙에 무한히 쌓임)    │
  └───────────────────────────────────────┘
                    │
                    ▼
              OutOfMemoryError
```

세마포어는 **동시 실행 수**를 제한할 뿐, **코루틴 생성 자체를 막지 않는다.** `receive()`는 ack와 무관하게 메시지를 계속 전달하므로, 처리 속도보다 유입 속도가 빠르면 대기 코루틴이 힙에 무한히 누적된다.

### 올바른 패턴 — bufferTimeout 배치 처리

```kotlin
private val consumeJob: Job = CoroutineScope(Dispatchers.Default).launch {
    messageBroker
        .consume("product-updated")
        .bufferTimeout(MAX_BUFFER_SIZE, Duration.ofSeconds(MAX_BUFFER_TIME))
        .asFlow()
        .flowOn(Dispatchers.IO)
        .collect { records ->
            withContext(NonCancellable) {
                records.last().receiverOffset().acknowledge()
                processRecords(records.toProductUpdateMessages())
            }
        }
}

@EventListener(ContextClosedEvent::class)
fun destroy() {
    runBlocking { consumeJob.cancelAndJoin() }
}
```

```
  bufferTimeout 동작 방식
  ───────────────────────

  receive() → [msg1] [msg2] [msg3] ... [msgN]
                │      │      │          │
                └──────┴──────┴────┬─────┘
                                   │
                    bufferTimeout(size=1000, time=5s)
                    → 1000개 모이거나 5초 경과 시 배치 전달
                                   │
                                   ▼
                         ┌─────────────────┐
                         │ List<Record>     │
                         │ [msg1..msg1000]  │
                         └────────┬────────┘
                                  │
                         processRecords(batch)
                                  │
                         last.acknowledge()
```

**핵심 변경점:**

1. **`bufferTimeout(size, duration)`**: 메시지를 모아서 배치 단위로 전달. 1000개가 모이거나 5초가 경과하면 리스트로 넘긴다
2. **`collect`로 순차 처리**: `launch`로 코루틴을 무한 생성하지 않고, `collect`가 하나의 배치 처리를 완료해야 다음 배치를 받는다
3. **마지막 offset만 acknowledge**: 배치의 마지막 레코드만 커밋하면 전체 배치가 처리된 것으로 기록된다
4. **`NonCancellable` 컨텍스트**: 처리 중 코루틴 취소가 발생해도 acknowledge까지 완료되도록 보장

배치 내부에서는 청크 단위로 나누어 세마포어 + `async`/`awaitAll`로 병렬 처리할 수 있다:

```kotlin
private suspend fun processRecords(messages: List<ProductUpdateMessage>) {
    val productNos = messages.map { it.productNo }.toSet()
    val chunks = productNos.chunked(CHUNK_SIZE)

    coroutineScope {
        val semaphore = Semaphore(MAX_CONCURRENCY)
        val jobs = chunks.map { chunk ->
            async {
                semaphore.acquire()
                try {
                    productIndexService.updateIndex(chunk.toSet())
                } finally {
                    semaphore.release()
                }
            }
        }
        jobs.awaitAll()
    }
}
```

여기서 세마포어는 **유한한 청크 목록** 안에서만 동시 실행 수를 제어하므로 안전하다. `bufferTimeout`이 먼저 총량을 제한하고, 그 안에서 청크를 나눠 병렬 처리하는 2단계 구조다.

```
  bufferTimeout(1000, 5s)
         │
         ▼
  ┌──────────────────────────────────────┐
  │  배치: 1000개 메시지                   │
  │  → productNo 중복 제거 → 예: 800개    │
  │  → chunked(2000) → [chunk1]          │
  └──────────────┬───────────────────────┘
                 │
                 ▼
  ┌──────────────────────────────────────┐
  │  Semaphore(15)                       │
  │  async { updateIndex(chunk1) }       │
  │  async { updateIndex(chunk2) }  ...  │
  │  → 최대 15개 청크만 동시 실행          │
  │  → awaitAll()로 전체 완료 대기         │
  └──────────────────────────────────────┘
```

### 실패 시 재시도 — Retry와 Dead Letter 패턴

배치 처리 중 실패하면 재시도 카운트를 증가시켜 같은 토픽에 다시 발행하고, 최대 재시도 횟수를 초과하면 실패 레코드를 별도 저장소에 기록한다:

```kotlin
private suspend fun processRecords(messages: List<ProductUpdateMessage>) {
    runCatching {
        updateProductIndex(messages)
    }.onFailure { e ->
        val retryable = messages.filter { it.retryCount < MAX_RETRY_COUNT }
        val failed = messages.filter { it.retryCount >= MAX_RETRY_COUNT }

        // 재시도 가능한 메시지는 카운트 증가 후 재발행
        if (retryable.isNotEmpty()) {
            messageBroker.send(
                topic = "product-updated",
                messages = retryable.map { it.copy(retryCount = it.retryCount + 1) }
            )
        }
        // 최대 재시도 초과 메시지는 실패 기록
        if (failed.isNotEmpty()) {
            failRecordRepository.save(
                FailRecord(topic = "product-updated", payload = failed, error = e.message)
            )
            log.error("Max retry exceeded. Saved to fail record.", e)
        }
    }
}
```

---

## 파티션 — 병렬 처리와 순서 보장의 트레이드오프

### 파티션의 역할

파티션은 토픽의 메시지를 **물리적으로 분리**해서 병렬 처리를 가능하게 한다.

```
  Topic: "product-updated" (3 partitions)
  ────────────────────────────────────────

  ┌──────────────────────┐
  │ Partition 0          │
  │ offset: 0,1,2,3,4   │──▶ Consumer A
  └──────────────────────┘

  ┌──────────────────────┐
  │ Partition 1          │
  │ offset: 0,1,2,3     │──▶ Consumer B
  └──────────────────────┘

  ┌──────────────────────┐
  │ Partition 2          │
  │ offset: 0,1,2       │──▶ Consumer C
  └──────────────────────┘

  ※ 각 파티션은 독립적인 offset을 가짐
  ※ 파티션 내부에서는 순서 보장 ✓
  ※ 파티션 간에는 순서 보장 ✗
```

> 📌 Apache Kafka 공식 문서:
> *"Kafka only provides a total order over records within a partition, not between different partitions in a topic."*
> — [Kafka Documentation: Topics and Logs](https://kafka.apache.org/documentation/#intro_topics)

### 파티션 간 순서 문제

상품 상태 변경처럼 **이벤트 순서**가 중요한 경우 문제가 된다:

```
  상품 등록 → 가격 변경 → 판매 중지
      │           │           │
      ▼           ▼           ▼
  Partition 0  Partition 1  Partition 2
  (Consumer A) (Consumer B) (Consumer C)

  → "판매 중지"가 "상품 등록"보다 먼저 처리될 수 있음!
```

인덱싱 컨슈머에서 이 순서가 뒤바뀌면, 판매 중지된 상품이 검색 결과에 노출되거나, 등록되지 않은 상품의 가격 변경이 무시되는 문제가 발생할 수 있다.

### 해결 — 파티션 키

카프카의 기본 분배 방식은 **Round Robin**이다. 하지만 **파티션 키를 지정하면 키의 해시값으로 파티션이 결정**된다.

```
  파티션 키 = productNo (상품 번호)
  ────────────────────────────────────

  productNo=12345 의 모든 이벤트
    상품등록 → 가격변경 → 판매중지
       │          │          │
       └──────────┴──────────┘
                  │
         hash(12345) % 3 = 1
                  │
                  ▼
            Partition 1  ← 같은 상품은 항상 같은 파티션
            (순서 보장 ✓)
```

```kotlin
// Producer: 파티션 키 지정
kafkaSender.createOutbound()
    .send(
        Flux.fromIterable(messages).map { message ->
            ProducerRecord(
                "product-updated",
                message.productNo.toString(),  // ← 파티션 키 = 상품 번호
                message.toJson()
            )
        }
    )
    .then()
    .subscribe()
```

> 📌 Apache Kafka 공식 문서:
> *"Records with the same key will be sent to the same partition."*
> — [Kafka Documentation: Producer](https://kafka.apache.org/documentation/#producerconfigs)

**같은 키를 가진 레코드는 항상 같은 파티션으로 분배되므로, 해당 키 범위 내에서 순서가 보장된다.**

### 파티션 키의 한계 — 데이터 편향

```
  productNo 1~1000   (인기 상품: 변경 10만건/일) ──▶ Partition 1 ███████████
  productNo 1001~5000 (일반 상품: 변경 1000건/일) ──▶ Partition 0 █
  productNo 5001~9999 (비활성 상품: 변경 50건/일) ──▶ Partition 2 ▌

  → 특정 파티션에 레코드가 몰리는 Hot Partition 문제
```

상품별 변경 빈도 차이가 크면 특정 파티션에 레코드가 집중되어 Consumer 간 부하가 불균형해진다. 이 경우 파티션 키 설계를 재고하거나, 파티션 수를 늘려서 분산시켜야 한다.

---

## 핵심 정리

| 개념 | 흔한 오해 | 실제 동작 |
|------|----------|----------|
| Consumer Group | 여러 컨슈머가 동시에 소비 | 파티션당 1 컨슈머만 활성, 나머지는 스탠바이 |
| Offset acknowledge | ack해야 다음 메시지를 읽음 | ack와 무관하게 poll 계속 진행. offset은 재시작 시 시작 위치 |
| 파티션 | 파티션을 늘리면 무조건 좋다 | 파티션 간 순서 미보장, 리밸런싱 시 복구 시간 증가 |
| 세마포어 + launch | 동시 처리 수를 제한한다 | 실행 수만 제한, 코루틴 생성(힙 점유)은 제한하지 않음 |
| bufferTimeout + collect | — | 배치 단위 수집 → 순차 소비로 백프레셔 보장 |

---

## 참고 자료

- [Apache Kafka Documentation — Design](https://kafka.apache.org/documentation/#design)
- [Apache Kafka Documentation — Consumer](https://kafka.apache.org/documentation/#theconsumer)
- [Reactor Kafka Reference Guide](https://projectreactor.io/docs/kafka/release/reference/)
- [Kotlin Coroutines — Semaphore](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-semaphore/)