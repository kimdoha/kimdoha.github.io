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

## 파티션이란?

카프카에서 **토픽(Topic)**은 메시지를 분류하는 논리적 채널이다. 그리고 **파티션(Partition)**은 그 토픽을 물리적으로 쪼갠 단위다.

```
  Topic: "product-updated"
  ┌──────────────────────────────────────────────────────┐
  │                                                      │
  │  Partition 0:  [msg0][msg1][msg2][msg3][msg4] →      │
  │  Partition 1:  [msg0][msg1][msg2] →                  │
  │  Partition 2:  [msg0][msg1][msg2][msg3] →            │
  │                                                      │
  └──────────────────────────────────────────────────────┘
  ※ 각 파티션은 독립된 메시지 큐
  ※ 파티션마다 자체 offset(순번)을 관리
```

하나의 토픽에 파티션이 3개면, 프로듀서가 보낸 메시지는 이 3개의 파티션 중 하나에 저장된다. 파티션 내부에서는 **메시지 순서가 보장**되지만, 서로 다른 파티션 간에는 순서가 보장되지 않는다.

파티션을 나누는 이유는 **병렬 처리**다. 파티션이 1개면 컨슈머도 1개만 동작할 수 있지만, 파티션이 3개면 최대 3개의 컨슈머가 동시에 메시지를 처리할 수 있다.

---

## Consumer Group — 병렬 소비와 리밸런싱

### 기본 구조

카프카의 Consumer Group은 **파티션 단위의 병렬 소비를 위한 수평 확장 메커니즘**이다. 같은 그룹에 속한 컨슈머들이 토픽의 파티션을 나눠 가지며, 각 파티션은 그룹 내 정확히 하나의 컨슈머에게만 배정된다.

```
  Topic: 3 partitions, Consumer Group: 3 consumers
  ─────────────────────────────────────────────────

  ┌────────────────────────────────────────────────────┐
  │         Consumer Group: "product-consumer-group"   │
  │                                                    │
  │   ┌──────────┐  ┌──────────┐  ┌──────────┐        │
  │   │Consumer A│  │Consumer B│  │Consumer C│        │
  │   │ (active) │  │ (active) │  │ (active) │        │
  │   └────┬─────┘  └────┬─────┘  └────┬─────┘        │
  │        │              │              │              │
  └────────┼──────────────┼──────────────┼─────────────┘
           │              │              │
      ┌────▼───┐    ┌────▼───┐    ┌────▼───┐
      │   P0   │    │   P1   │    │   P2   │
      └────────┘    └────────┘    └────────┘

  ※ 파티션 수 = 컨슈머 수 → 모든 컨슈머가 활성 상태
  ※ 각 컨슈머가 담당 파티션의 메시지를 독립적으로 처리
```

핵심 규칙:

- 같은 컨슈머 그룹 안에서, **각 파티션은 정확히 하나의 컨슈머에게만 배정**된다
- 파티션 수만큼 컨슈머가 **동시에 활성** 상태로 병렬 처리한다
- 활성 컨슈머에 장애가 발생하면 해당 파티션이 다른 컨슈머에게 재배정된다 (**리밸런싱**)

> 📌 Confluent 공식 문서:
> *"A consumer group is a set of consumers from the same application that work together to consume and process messages from one or more topics."*
> — [Confluent: Consumer Design](https://docs.confluent.io/kafka/design/consumer-design.html)

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
// ⚠️ 개념 설명용 의사코드 — 실제 코드는 Reactor Kafka 기반 (Spring @KafkaListener 미사용)

// 상품 인덱싱 컨슈머
@KafkaListener(
    topics = ["product-updated"],
    groupId = "product-indexing-group"
)
fun consumeForIndexing(record: ConsumerRecord<String, String>) {
    productIndexService.updateIndex(record.value())
}

// 상품 웹훅 알림 컨슈머 — 같은 토픽이지만 반드시 다른 group.id
@KafkaListener(
    topics = ["product-updated"],
    groupId = "product-webhook-group"
)
fun consumeForWebhook(record: ConsumerRecord<String, String>) {
    webhookService.notify(record.value())
}
```

같은 토픽을 구독하는 두 컨슈머가 같은 `group.id`를 사용하면, 카프카는 이들을 하나의 그룹으로 인식하고 파티션을 나눠버린다. 각 컨슈머가 **전체 메시지를 모두 받아야** 하는 상황에서 일부만 받게 되는 문제가 발생한다.

---

## Offset — 가장 흔한 오해

### 오해: ack를 보내야 다음 메시지를 읽는다

카프카의 `acknowledge()`를 RabbitMQ의 ack처럼 생각하기 쉽다. **하지만 카프카의 offset commit은 메시지 흐름을 제어하지 않는다.**

이 오해는 RabbitMQ와 Kafka의 메시지 전달 모델이 근본적으로 다르기 때문에 생긴다:

| | RabbitMQ | Kafka |
|---|---|---|
| 전달 방식 | **Push** — 브로커가 컨슈머에게 전달 | **Pull** — 컨슈머가 `poll()`로 가져감 |
| ack의 역할 | **"다음 메시지를 보내도 된다"는 신호** — ack 없으면 브로커가 전달을 멈춤 | **"여기까지 처리했다"는 북마크** — commit 없어도 poll은 계속 진행 |
| ack 안 하면? | `prefetch` 한도에 도달하면 브로커가 **전송 중단** | 아무 영향 없음. 단, 재시작 시 **이미 처리한 메시지를 다시 읽음** |

```
  Kafka: ack(offset commit) 안 했을 때 무슨 일이 일어나는가?
  ──────────────────────────────────────────────────────────

  [정상 운영 중] — 아무 영향 없음
  ─────────────────────────────────

  Broker Partition:  [msg0][msg1][msg2][msg3][msg4][msg5]
                                        ▲               ▲
                              committed offset = 3    position = 6
                              (acknowledge 한 지점)   (poll이 읽고 있는 지점)

  Consumer:
    poll() → msg3 처리 ✓ → acknowledge(msg3) → committed offset = 3
    poll() → msg4 처리 ✓ → (acknowledge 안 함)
    poll() → msg5 처리 ✓ → (acknowledge 안 함)
                             │
                    poll은 멈추지 않는다.
                    position은 6으로 계속 전진.
                    committed offset만 3에 머물러 있음.

  ─────────────────────────────────────────────────────────

  [장애 발생 → 재시작] — 이미 처리한 메시지를 다시 읽음
  ─────────────────────────────────────────────────────────

  Consumer 프로세스 종료 → position 소멸 (메모리에만 존재)

  재시작 시:
    "마지막으로 커밋한 offset이 어디지?" → 3
    "그럼 offset 3부터 다시 읽자"

  Broker Partition:  [msg0][msg1][msg2][msg3][msg4][msg5]
                                        ▲
                              committed offset = 3
                              ← 여기부터 다시 시작

  Consumer (재시작):
    poll() → msg3 처리 (중복!) ← 이미 처리했지만 커밋 안 해서 다시 읽음
    poll() → msg4 처리 (중복!)
    poll() → msg5 처리 (중복!)

  ─────────────────────────────────────────────────────────

  핵심: offset commit은 "다음 메시지를 받기 위한 조건"이 아니라
       "재시작 시 어디서부터 읽을지 저장하는 북마크"다.
```

이 동작 차이가 발생하는 이유는 컨슈머 내부에 **독립적인 포인터 2개**가 존재하기 때문이다:

```
  position과 committed offset — 두 개의 독립된 포인터
  ───────────────────────────────────────────────────

  ┌──────────────────────────────────────────────┐
  │  Consumer (JVM 프로세스 메모리)                │
  │                                                │
  │  position = 6                                  │
  │  → poll()이 다음에 읽을 위치                    │
  │  → poll() 호출마다 자동 전진                    │
  │  → 메모리에만 존재 (프로세스 종료 시 소멸)       │
  └──────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────┐
  │  Kafka Broker (__consumer_offsets 토픽)        │
  │                                                │
  │  committed offset = 3                          │
  │  → acknowledge()/commit() 호출 시에만 갱신     │
  │  → 브로커에 영구 저장 (프로세스 종료 후에도 유지) │
  └──────────────────────────────────────────────┘

  정상 운영 중 두 포인터는 서로를 참조하지 않는다:

  시간 →
  poll()     poll()     ack(msg2)     poll()     poll()
    │          │            │           │          │
    ▼          ▼            │           ▼          ▼
  position:  0 → 1 → 2     │      → 3 → 4 → 5
                            │
                            ▼
  committed:  0    0    0 → 2      2    2    2

  position은 poll()이 전진시키고,
  committed는 acknowledge()가 전진시킨다.
  정상 운영 중에는 committed offset을 아무도 읽지 않는다.
```

committed offset이 position에 영향을 주는 시점은 **시작/리밸런싱뿐**이다:

```
  committed offset → position 초기화가 일어나는 2가지 시점
  ──────────────────────────────────────────────────────

  1. 컨슈머 시작 (또는 재시작)
  ────────────────────────────
     position이 없음 (메모리가 비어있음)
     → committed offset을 읽어서 position 초기값으로 설정
     → 이후부터는 position이 독립적으로 전진

  2. 리밸런싱으로 새 파티션을 할당받았을 때
  ─────────────────────────────────────────
     해당 파티션의 position이 없음
     → committed offset을 읽어서 position 초기값으로 설정

  요약:
    position → committed:  acknowledge() 호출 시 (현재 위치를 브로커에 저장)
    committed → position:  시작/리밸런싱 시에만 (저장된 위치에서 position 초기화)
    정상 운영 중:           서로 독립. 각자 움직임.
```

> 📌 Kafka 공식 JavaDoc ([KafkaConsumer](https://kafka.apache.org/25/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html)):
> - **position**: *"the offset of the next record that will be given out. It **automatically advances** every time the consumer receives messages in a call to `poll()`."*
> - **committed offset**: *"the last offset that has been stored securely. Should the process **fail and restart**, this is the offset that the consumer will recover to."*
>
> position(다음 읽을 위치)은 poll마다 자동 전진하고, committed offset(커밋된 위치)은 장애 복구 시에만 참조된다. 두 개념이 완전히 분리되어 있다.

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
  │  acknowledge(msg4)  → committed offset = 5로 갱신    │
  │  → 다음 재시작 시 msg5부터 시작                        │
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
                processRecords(records.toProductUpdateMessages())
                records.last().receiverOffset().acknowledge()
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
3. **마지막 offset만 acknowledge**: 배치의 마지막 레코드만 커밋하면 전체 배치가 처리된 것으로 기록된다. 단, 이 방식은 **단일 파티션 배치**에서만 안전하다. `bufferTimeout`이 여러 파티션의 레코드를 하나의 배치로 묶을 수 있으므로, 멀티 파티션 환경에서는 파티션별 마지막 레코드를 각각 acknowledge해야 한다
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

이벤트 내용을 직접 적용하는 컨슈머라면, 이 순서가 뒤바뀔 때 판매 중지된 상품이 검색 결과에 노출되거나, 등록되지 않은 상품의 가격 변경이 무시되는 문제가 발생할 수 있다.

**단, 현재 인덱싱 컨슈머(`ProductIndexingConsumer`)는 이 문제의 실질적 영향이 제한적이다.** 이벤트 메시지에는 `productNo`만 포함되어 있고, 컨슈머는 이를 트리거로 DB에서 최신 상태를 다시 조회하여 인덱싱한다:

```kotlin
// ProductIndexService.getIndexParameters()
val products: List<Product> = coroutineTransactionHandler.runReadOnlyTransaction {
    productRepository.findProductsByProductNoIn(productNos)  // DB에서 최신 상태 조회
}
```

"판매 중지" 이벤트보다 "상품 등록" 이벤트가 나중에 처리되더라도, 그 시점에 DB의 상태는 이미 "판매 중지"이므로 인덱싱 결과는 올바르다. 이벤트는 **"무엇이 변경됐는지"를 알리는 트리거**이지, **변경 내용 자체를 전달하지 않기 때문**이다.

이처럼 이벤트를 트리거로만 사용하는 패턴에서는 파티션 키 없이도 최종 일관성이 유지된다. 반면, **이벤트 내용을 직접 적용하는 컨슈머**(예: 이벤트의 상태값으로 덮어쓰는 방식)에서는 파티션 키를 반드시 지정해야 한다.

### 해결 — 파티션 키

파티션 키가 없는 메시지의 기본 분배 방식은 Kafka 버전에 따라 다르다:

- **Kafka 2.3 이하**: Round Robin — 메시지를 파티션에 하나씩 돌아가며 분배
- **Kafka 2.4 이상**: **Sticky Partitioner** ([KIP-480](https://cwiki.apache.org/confluence/display/KAFKA/KIP-480:+Sticky+Partitioner)) — 하나의 파티션에 배치가 찰 때까지 붙어서 전송한 뒤 다른 파티션으로 전환. 배치 효율이 높아져 지연 시간이 줄어든다

**파티션 키를 지정하면** 버전과 무관하게 **키의 해시값으로 파티션이 결정**된다.

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

현재 코드는 파티션 키를 사용하지 않는다. 모든 `ProducerRecord`가 `(topic, value)` 2개 인자로만 생성되어 기본 파티셔닝 전략을 따른다:

```kotlin
// 현재 코드 (CommonKafkaManager.kt) — 파티션 키 없음
ProducerRecord(topic.value, it.toJson())  // (topic, value) — key = null
```

이벤트 내용을 직접 적용하는 컨슈머에서 순서 보장이 필요하다면, 다음과 같이 파티션 키를 지정할 수 있다:

```kotlin
// 파티션 키 지정 예시
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
> *"If a valid partition number is specified that partition will be used when sending the record. If no partition is specified but a key is present a partition will be chosen using a hash of the key."*
> — [Kafka Documentation: ProducerRecord](https://kafka.apache.org/documentation/#producerapi)

**같은 키를 가진 레코드는 항상 같은 파티션으로 분배되므로, 해당 키 범위 내에서 순서가 보장된다.** 단, 이는 **파티션 수가 변하지 않을 때만** 성립한다. `hash(key) % numPartitions`로 파티션이 결정되므로, 파티션 수를 변경하면 기존 키의 파티션 배정이 달라질 수 있다.

### 파티션 키의 한계 — 데이터 편향

```
  productNo=100  (인기 상품: 변경 10만건/일)
  productNo=200  (인기 상품: 변경 8만건/일)
      │
      │  hash(100) % 3 = 1
      │  hash(200) % 3 = 1   ← 해시 충돌로 같은 파티션에 배정
      ▼
  Partition 0:  █           (변경 1000건/일)
  Partition 1:  ███████████ (변경 18만건/일) ← Hot Partition
  Partition 2:  ▌           (변경 50건/일)

  → 특정 키들의 해시값이 같은 파티션에 몰리는 Hot Partition 문제
```

특정 파티션 키의 트래픽이 극단적으로 많거나, 해시 충돌로 고빈도 키들이 같은 파티션에 몰리면 Consumer 간 부하가 불균형해진다. 이 경우 파티션 키 설계를 재고하거나, 파티션 수를 늘려서 분산시켜야 한다.

---

## 핵심 정리

| 개념 | 흔한 오해 | 실제 동작 |
|------|----------|----------|
| Consumer Group | 같은 파티션을 그룹 내 여러 컨슈머가 동시에 읽을 수 있다 | 같은 그룹 내에서 각 파티션은 정확히 하나의 컨슈머에게만 배정된다 |
| Offset acknowledge | ack해야 다음 메시지를 읽음 | ack와 무관하게 poll 계속 진행. offset은 재시작 시 시작 위치 |
| 파티션 | 파티션을 늘리면 무조건 좋다 | 파티션 간 순서 미보장, 리밸런싱 시 복구 시간 증가 |
| 세마포어 + launch | 동시 처리 수를 제한한다 | 실행 수만 제한, 코루틴 생성(힙 점유)은 제한하지 않음 |
| bufferTimeout + collect | — | 배치 단위 수집 → 순차 소비로 백프레셔 보장 |

---