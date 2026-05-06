---
title: "카프카 톺아보기 (2) — Rebalancing의 원리와 장애 복구 전략"
date: 2026-05-06 10:00:00 +0900
categories: [Kafka]
tags: [kafka, rebalancing, consumer-group, static-membership, cooperative-rebalancing]
toc: true
---

---

## 들어가며

[1편]({% post_url 2026-05-02-kafka-lessons-from-failures %})에서 Consumer Group·Offset·파티션의 실제 동작을 다뤘다. 이번 글에서는 **컨슈머가 그룹에서 쫓겨나는 원인**, **리밸런싱 프로토콜의 단계별 동작**, 그리고 **장애를 최소화하는 개선 전략**을 정리한다.

---

## 리밸런싱이란?

Consumer Group 내의 **파티션-컨슈머 매핑을 재조정**하는 과정이다. 다음 상황에서 발생한다:

- 컨슈머가 그룹에서 이탈 (장애, 타임아웃)
- 새로운 컨슈머가 그룹에 합류
- 구독 중인 토픽의 파티션 수가 변경

```
  리밸런싱 전                    리밸런싱 후
  ──────────────                ──────────────
  P0 → Consumer A               P0 → Consumer A
  P1 → Consumer B               P1 → Consumer A  ← 재배정
  P2 → Consumer C (장애)        P2 → Consumer B  ← 재배정
  P3 → Consumer D               P3 → Consumer D

  ※ Consumer C 이탈 → 남은 3명(A,B,D)에게 재배정 (Eager 프로토콜 기준)
  ※ 리밸런싱 중에는 모든 컨슈머의 소비가 멈춤 (→ 개선 방안은 후술)
```

문제는 **리밸런싱 중에 전체 컨슈머의 메시지 소비가 멈춘다**는 것이다. 이를 **Stop-the-World**라고 부른다.

---

## 컨슈머가 쫓겨나는 원인

컨슈머가 그룹에서 이탈하는 원인은 크게 두 가지다.

### Case 1: session.timeout.ms 초과 — 브로커가 감지

```
  Consumer                       Broker (Group Coordinator)
     │                               │
     ├── heartbeat ──────────────────▶│  ← 정상
     │          (heartbeat.interval.ms 간격)
     │                               │
     ├── heartbeat ──────────────────▶│  ← 정상
     │                               │
     │   ┌───────────────────────┐   │
     │   │ 네트워크 장애 / GC    │   │
     │   │ heartbeat 전송 불가   │   │
     │   └───────────────────────┘   │
     │                               │
     │         ··· 시간 경과 ···       │
     │                               │
     │                               ├── session.timeout.ms 초과 감지
     │                               ├── "이 컨슈머는 죽었다" 판단
     │                               ├── 그룹에서 강제 제거
     │                               └── 나머지 컨슈머에게 리밸런싱 통보
```

브로커가 `session.timeout.ms` 동안 heartbeat를 받지 못하면, 해당 컨슈머를 **강제로 그룹에서 제거**한다. 컨슈머는 LeaveGroup 요청을 보내지 않는다 — 브로커 단독 판단이다.

> 📌 Apache Kafka 공식 문서:
> *"If no heartbeats are received by the broker before the expiration of this session timeout, then the broker will remove the worker from the group and initiate a rebalance."*
> — [Kafka Consumer Configs: session.timeout.ms](https://kafka.apache.org/documentation/#consumerconfigs_session.timeout.ms)

### Case 2: max.poll.interval.ms 초과 — 클라이언트가 감지

```
  Consumer                       Broker (Group Coordinator)
     │                               │
     ├── poll() 호출 ─────────────────│
     │                               │
     │   ┌──────────────────────────┐│
     │   │ 처리 로직이 너무 오래 걸림 ││
     │   │ (대량 배치, 외부 API 지연) ││
     │   └──────────────────────────┘│
     │                               │
     │         ··· 시간 경과 ···       │
     │                               │
     ├── max.poll.interval.ms 초과   │
     │   (클라이언트 스스로 감지)      │
     │                               │
     ├── LeaveGroup 능동 전송 ───────▶│  ← 클라이언트가 먼저 떠남
     │                               ├── 리밸런싱 시작
```

`max.poll.interval.ms`(기본값: 5분) 동안 `poll()`이 호출되지 않으면, **클라이언트 스스로** 그룹을 떠난다. 이때 로그에는 다음과 같이 남는다:

```
consumer pro-actively leaving the group
```

```
CommitFailedException: Offset commit cannot be completed since the consumer
is not part of an active group for auto partition assignment;
it is likely that the consumer was kicked out of the group.
```

> 📌 Apache Kafka 공식 문서:
> *"If poll() is not called before expiration of this timeout, then the consumer is considered failed and the group will rebalance to reassign the partitions to another member."*
> — [Kafka Consumer Configs: max.poll.interval.ms](https://kafka.apache.org/documentation/#consumerconfigs_max.poll.interval.ms)

### 두 케이스의 차이 정리

| | session.timeout.ms 초과 | max.poll.interval.ms 초과 |
|---|---|---|
| **감지 주체** | 브로커 | 클라이언트 |
| **이탈 방식** | 브로커가 강제 제거 | 클라이언트가 능동적으로 LeaveGroup 전송 |
| **원인** | heartbeat 미전송 (네트워크, GC 등) | 처리 로직 지연 (대량 배치, 외부 호출) |

---

## assign()과 subscribe() — 왜 이전에는 문제가 없었는가

Kafka 컨슈머의 파티션 할당 방식은 두 가지다.

### subscribe() — 브로커가 관리

```kotlin
// 브로커가 파티션-컨슈머 매핑을 관리
consumer.subscribe(listOf("product-updated"))
```

- 브로커의 Group Coordinator가 파티션 배정을 중재 (실제 할당은 리더 컨슈머가 수행)
- `session.timeout.ms` / `max.poll.interval.ms` 초과 시 **리밸런싱 발생**
- 컨슈머 추가/제거 시 자동 재배정

### assign() — 수동 지정

```kotlin
// 개발자가 직접 파티션을 지정
consumer.assign(listOf(TopicPartition("product-updated", 0)))
```

- 브로커의 그룹 관리 기능을 **사용하지 않음**
- `max.poll.interval.ms`를 초과해도 **리밸런싱이 발생하지 않음**
- 장애 시 자동 재배정 없음 — 수동 복구 필요

> 📌 Apache Kafka 공식 문서:
> *"Manual partition assignment through this method does not use the consumer's group management functionality. As such, there will be no rebalance operation triggered when group membership or cluster and topic metadata change."*
> — [KafkaConsumer Javadoc: assign()](https://kafka.apache.org/38/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html)

```
  assign() 사용 시                subscribe() 사용 시
  ──────────────────              ──────────────────────
  ┌──────────┐                    ┌──────────────────────┐
  │ Consumer │                    │ Group Coordinator    │
  │ 직접 P0  │                    │ (Broker)             │
  │ 지정     │                    │                      │
  └────┬─────┘                    │ P0 → Consumer A      │
       │                          │ P1 → Consumer B      │
       ▼                          │ P2 → Consumer C      │
  ┌────────┐                      └──────────┬───────────┘
  │   P0   │                                 │
  └────────┘                      timeout 초과 시 리밸런싱
                                  ※ 자동 재배정
  ※ timeout 초과해도
    리밸런싱 없음
  ※ 장애 시 수동 복구
```

**기존에 assign()을 사용하던 서비스가 subscribe()로 전환하면**, 이전에는 발생하지 않던 리밸런싱이 처음으로 나타날 수 있다. `max.poll.interval.ms` 초과가 assign() 시절에는 무시되었지만, subscribe()에서는 컨슈머 이탈 + 전체 리밸런싱으로 이어진다.

> 📎 우리 서비스의 경우, 모든 컨슈머가 Reactor Kafka의 `ReceiverOptions.subscription(listOf(topic))` → `KafkaReceiver.create().receive()` 패턴을 사용하고 있다. 이는 내부적으로 subscribe() 방식이므로, 리밸런싱 대상이다.

---

## Eager 리밸런싱 프로토콜 — 단계별 동작

전통적인 리밸런싱 프로토콜(Eager)의 전체 흐름을 단계별로 살펴보자.

Kafka 3.1+부터 기본 `partition.assignment.strategy`가 `[RangeAssignor, CooperativeStickyAssignor]`로 변경되었지만, 리더가 **목록의 첫 번째 공통 전략**을 선택하기 때문에 `RangeAssignor`(Eager)가 우선 적용된다. Cooperative를 실제로 사용하려면 `CooperativeStickyAssignor`를 **명시적으로** 설정해야 한다.

### 1단계: 컨슈머 이탈

```
  ┌──────────────────────────────────────────────────┐
  │            Consumer Group (정상 상태)              │
  │                                                    │
  │  Consumer A ◄──► P0     (heartbeat 정상)           │
  │  Consumer B ◄──► P1     (heartbeat 정상)           │
  │  Consumer C ◄──► P2     (max.poll.interval 초과!)  │
  │                                                    │
  └──────────────────────────────────────────────────┘
                         │
                         ▼
  Consumer C: "poll() 간격이 5분을 넘었다. 그룹을 떠난다."
              → LeaveGroup Request 전송
              → generation/memberID 초기화
```

### 2단계: 나머지 컨슈머에게 통보

```
  Consumer A                    Broker (Group Coordinator)
     │                               │
     ├── heartbeat ──────────────────▶│
     │                               │
     │◀── REBALANCE_IN_PROGRESS ─────┤  ← heartbeat 응답으로 통보
     │                               │
     │  "리밸런싱이 시작되었다.        │
     │   현재 파티션을 모두 반납하라." │
     │                               │
     ├── 파티션 P0 반납              │
     │   (onPartitionsRevoked 호출)  │
```

**핵심**: 나머지 컨슈머는 다음 heartbeat **응답**에서 `REBALANCE_IN_PROGRESS` 에러 코드를 받아 리밸런싱을 인지한다.

### 3단계: Stop-the-World

```
  ┌──────────────────────────────────────────────────┐
  │              Stop-the-World 구간                   │
  │                                                    │
  │  Consumer A: 파티션 반납 완료, 소비 중단 ■          │
  │  Consumer B: 파티션 반납 완료, 소비 중단 ■          │
  │  Consumer C: 이미 이탈                             │
  │                                                    │
  │  → 어떤 컨슈머도 메시지를 처리하지 않는 구간         │
  │  → 파티션이 많을수록 이 구간이 길어짐               │
  └──────────────────────────────────────────────────┘
```

Eager 프로토콜에서는 **모든 컨슈머가 기존 파티션을 반납**한 뒤에야 재배정이 시작된다. 이 구간 동안 전체 메시지 소비가 멈춘다.

### 4단계: JoinGroup — 그룹 재구성

```
  Consumer A                    Broker                    Consumer B
     │                            │                          │
     ├── JoinGroup Request ──────▶│◀── JoinGroup Request ───┤
     │   - group.id               │    - group.id            │
     │   - member.id              │    - member.id           │
     │   - session.timeout        │    - session.timeout     │
     │   - rebalance.timeout      │    - rebalance.timeout   │
     │     (= max.poll.interval.ms)│     (= max.poll.interval.ms)│
     │                            │                          │
     │                            │  (모든 멤버 도착 대기)    │
     │                            │                          │
     │◀── JoinGroup Response ─────┤──── JoinGroup Response ─▶│
     │   - leader: Consumer A     │    - leader: Consumer A  │
     │   - members: [A, B]        │    - members: []         │
     │     (리더만 전체 목록 수신)  │      (일반 멤버는 빈 배열) │
```

> 📌 Apache Kafka: Client-side Assignment Proposal (요약)
> *"The JoinGroup response includes an array for the members of the group along with their metadata. This is only populated for the leader to reduce the overall overhead of the protocol; for other members, it will be empty."*

**리더 컨슈머**만 전체 멤버 목록을 수신하고, **로컬에서** 파티션 할당을 수행한다. 브로커는 할당을 중재하지 않는다.

### 5단계: SyncGroup — 할당 결과 배포

```
  Consumer A (리더)             Broker                    Consumer B
     │                            │                          │
     │  (로컬에서 파티션 할당 수행) │                          │
     │  A → [P0, P1]              │                          │
     │  B → [P2]                  │                          │
     │                            │                          │
     ├── SyncGroup Request ──────▶│◀── SyncGroup Request ───┤
     │   assignments:             │    assignments: {}       │
     │   { A:[P0,P1], B:[P2] }   │    (리더만 할당 전달)     │
     │                            │                          │
     │◀── SyncGroup Response ─────┤──── SyncGroup Response ─▶│
     │   your partitions: [P0,P1] │    your partitions: [P2] │
     │                            │                          │
     ├── 소비 재개 ✓              │              소비 재개 ✓ ─┤
     │   (onPartitionsAssigned)   │    (onPartitionsAssigned) │
```

SyncGroup 응답을 받은 각 컨슈머는 `onPartitionsAssigned()` 콜백이 호출되며 메시지 소비를 재개한다.

### 6단계: 이탈 컨슈머 복구 시 — 또다시 리밸런싱

```
  ┌──────────────────────────────────────────────────┐
  │  A:[P0,P1]  B:[P2]  (안정 상태)                    │
  └──────────────────────────────────────────────────┘
                         │
                Consumer C 복구 → JoinGroup Request
                         │
                         ▼
  ┌──────────────────────────────────────────────────┐
  │         또다시 Stop-the-World 발생!                │
  │                                                    │
  │  A: 파티션 반납, 소비 중단 ■                         │
  │  B: 파티션 반납, 소비 중단 ■                         │
  │  C: 합류 대기                                       │
  │                                                     │
  │  → JoinGroup → SyncGroup → 3명으로 재배정            │
  └──────────────────────────────────────────────────┘
                         │
                         ▼
           A:[P0]  B:[P1]  C:[P2]  (새 안정 상태)
```

Eager 프로토콜에서는 **컨슈머가 떠날 때 1번, 복구될 때 1번**, 총 2번의 Stop-the-World가 발생한다.

---

## Eager 프로토콜의 문제점 정리

```
  시간 ──────────────────────────────────────────────────▶

  정상 운영 │■■■ STW ■■■│ 재배정 운영 │■■■ STW ■■■│ 최종 운영
            │           │             │           │
         C 이탈      A,B 재배정     C 복구      A,B,C 재배정
            │           │             │           │
            └── 전체    └── 전체      └── 전체    └── 전체
                소비 중단   소비 재개      소비 중단    소비 재개

  ※ 파티션이 많을수록 STW 구간이 길어짐
  ※ 컨슈머가 불안정하면 연쇄 리밸런싱 가능
```

---

## 개선 방안 1: Static Membership (KIP-345, Kafka 2.3+)

### 문제

컨슈머가 일시적으로 오류를 겪을 때마다 LeaveGroup → 전체 리밸런싱이 발생한다.

### 해결: group.instance.id 설정

```kotlin
val props = Properties().apply {
    put(ConsumerConfig.GROUP_INSTANCE_ID_CONFIG, "consumer-instance-1")
    // 각 컨슈머 인스턴스마다 고유한 값 설정
}
```

```
  일반 멤버십 (Dynamic)              정적 멤버십 (Static)
  ────────────────────              ────────────────────
  Consumer C 처리 지연               Consumer C 일시 장애
  (max.poll.interval 초과)
       │                                 │
       ▼                                 ▼
  LeaveGroup 전송                   LeaveGroup 전송 안 함
       │                                 │
       ▼                                 ▼
  전체 리밸런싱 (STW)               다른 컨슈머는 계속 소비 ✓
       │                                 │
       ▼                                 ▼
  C 복구 → 또 리밸런싱 (STW)        C 복구 → 기존 파티션 그대로 이어받음
                                    (브로커에 캐싱된 instance.id 사용)
```

> 📌 KIP-345:
> *"Static members will not send leave group request when they go offline."*
> — [KIP-345: Introduce static membership protocol](https://cwiki.apache.org/confluence/display/KAFKA/KIP-345%3A+Introduce+static+membership+protocol+to+reduce+consumer+rebalances)

### 동작 원리

1. 컨슈머가 `group.instance.id`를 설정하면, 브로커가 이 ID를 캐싱
2. 일시적 장애 시 LeaveGroup을 **전송하지 않음**
3. `session.timeout.ms` 만료 전에 복구되면 리밸런싱 없이 기존 파티션을 그대로 이어받음
4. `session.timeout.ms`까지 복구되지 않으면 그때서야 리밸런싱 발생

**운영 팁**: Static Membership 사용 시 `session.timeout.ms`를 충분히 길게 설정해야 한다 (예: 5~10분). 짧으면 일시 장애에도 리밸런싱이 발생해 Static Membership의 효과가 없다.

---

## 개선 방안 2: Cooperative Rebalancing (KIP-429, Kafka 2.4+)

### Eager의 근본 문제

Eager 프로토콜은 리밸런싱 시 **모든 컨슈머가 모든 파티션을 반납**한다. Consumer C 하나가 이탈했을 뿐인데, 정상 동작 중인 A와 B도 파티션을 반납해야 한다.

### 해결: 이동이 필요한 파티션만 재배정

```
  Eager 프로토콜                   Cooperative 프로토콜
  ──────────────                   ─────────────────────
  C 이탈                           C 이탈
     │                                │
     ▼                                ▼
  A: P0 반납 ■                     A: P0 유지, 소비 계속 ✓
  B: P1 반납 ■                     B: P1 유지, 소비 계속 ✓
  C: P2 이미 없음                  C: P2만 재배정 대상
     │                                │
     ▼                                ▼
  전체 재배정                       P2는 소유자 없음 → 바로 재배정 가능
  A→[P0,P1], B→[P2]               리밸런싱 1회: P2를 A에게 배정
     │                                │
     ▼                                ▼
  전체 소비 재개                    A: P0 계속 + P2 추가
                                   B: P1 계속
                                   ※ P0, P1은 한 번도 중단되지 않음
```

> 📌 KIP-429 (요약):
> 기존 Eager 프로토콜은 단일 리밸런싱의 동기화 장벽에 의존하여 모든 컨슈머가 파티션을 반납한 뒤 새 세대로 합류했다. Cooperative 프로토콜은 이를 연속 리밸런싱으로 대체하여, 이동이 필요한 파티션만 단계적으로 재배정한다.
> *"instead of relying on the single rebalance's synchronization barrier ... we use consecutive rebalances"*
> — [KIP-429: Kafka Consumer Incremental Rebalance Protocol](https://cwiki.apache.org/confluence/display/KAFKA/KIP-429%3A+Kafka+Consumer+Incremental+Rebalance+Protocol)

### 설정 방법

Kafka 3.1+ 기본값은 `[RangeAssignor, CooperativeStickyAssignor]`이지만, 리더가 목록 순서대로 선택하므로 `RangeAssignor`(Eager)가 우선된다. Cooperative를 사용하려면 명시적으로 설정해야 한다:

```kotlin
val props = Properties().apply {
    put(
        ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
        CooperativeStickyAssignor::class.java.name
    )
}
```

운영 중인 컨슈머 그룹에서 전환할 때는 **2단계 롤링 배포**가 필요하다:

1. **1차 배포**: 기존 전략과 함께 설정 → `[RangeAssignor, CooperativeStickyAssignor]`
2. **2차 배포**: `CooperativeStickyAssignor`만 남기기

한 번에 전환하면 프로토콜 불일치로 기존 컨슈머가 그룹 합류에 실패한다.

### 주의: 완전한 무중단은 아니다

Cooperative 프로토콜은 **파티션 이동이 필요한 경우** 내부적으로 2회 연속 리밸런싱을 수행한다:

1. **1차**: 어떤 파티션을 revoke할지 결정. 기존 소유자에게 해당 파티션만 반납 요청
2. **2차**: revoke된 파티션을 새 컨슈머에게 할당

단, 이탈한 컨슈머의 파티션처럼 **소유자가 없는 파티션**은 revoke가 불필요하므로 1차에서 바로 할당될 수 있다. 이동 대상 파티션만 순간 중단되고, 나머지는 계속 소비한다. "전체 중단"이 "부분 중단"으로 개선된 것이다.

---

## 개선 방안 정리: 버전별 가용 옵션

| Kafka 버전 | 기능 | 효과 |
|-----------|------|------|
| 2.3 미만 | Eager 프로토콜만 가능 | 리밸런싱 시 전체 STW |
| **2.3+** | **Static Membership** (KIP-345) | 일시 장애 시 리밸런싱 방지 |
| **2.4+** | **Cooperative Rebalancing** (KIP-429) | STW 범위를 이동 파티션으로 축소 |
| **3.1+** | 기본값에 CooperativeStickyAssignor 포함 | 단, RangeAssignor가 우선 → **명시 설정 필요** |
| 2.4+ | Static + Cooperative 조합 | 최선의 조합 — 일시 장애 무리밸런싱 + 리밸런싱 시 부분 중단 |

---

## 컨슈머 장애 복구 전략

리밸런싱을 개선하더라도, **컨슈머 자체의 에러 복구**는 별도로 처리해야 한다. Reactor Kafka와 일반 Kafka 각각의 복구 패턴을 살펴보자.

### Reactor Kafka — 파티션 이탈 감지 후 재구독

Reactor Kafka에서 컨슈머 처리 중 예외가 발생하면 Flux 체인이 끊어지면서 파티션에서 이탈할 수 있다. 이를 감지하고 자동 재구독하는 패턴:

```kotlin
@Component
class ProductIndexingConsumer(
    private val messageBroker: MessageBroker,
    private val applicationEventPublisher: ApplicationEventPublisher,
) {
    private var consumeJob: Job? = null

    @PostConstruct
    fun init() {
        startConsume()
    }

    private fun startConsume() {
        consumeJob = CoroutineScope(Dispatchers.Default).launch {
            messageBroker
                .consume("product-updated")
                .doFinally {
                    // Flux 종료 시(에러, 완료, cancel 모두) 재구독 이벤트 발행
                    applicationEventPublisher.publishEvent(
                        ResubscribeEvent(this@ProductIndexingConsumer)
                    )
                }
                .bufferTimeout(MAX_BUFFER_SIZE, Duration.ofSeconds(MAX_BUFFER_TIME))
                .asFlow()
                .collect { records ->
                    withContext(NonCancellable) {
                        processRecords(records.toProductUpdateMessages())
                        records.last().receiverOffset().acknowledge()
                    }
                }
        }
    }

    // 재구독 이벤트를 받으면 컨슈머를 다시 시작
    @EventListener
    fun onResubscribe(event: ResubscribeEvent) {
        consumeJob?.cancel()
        startConsume()
    }
}
```

```
  정상 소비 중
  ────────────
  consume() → bufferTimeout → collect → process → ack
                                          │
                              예외 발생! ──┘
                                          │
                                          ▼
                              Flux 종료 (doFinally)
                                          │
                                          ▼
                              ResubscribeEvent 발행
                                          │
                                          ▼
                              onResubscribe() 호출
                                          │
                                          ▼
                              startConsume() → 파티션에 재연결
```

> 📎 위 `ResubscribeEvent` 패턴은 개념 설명용 예시다. 우리 서비스에서는 이 패턴을 사용하지 않으며, 실제로는 다음과 같은 에러 복구 방식을 사용한다:
> - **product**: 실패 레코드를 MongoDB에 저장 (`KafkaFailRecord`) → 수동/배치 재처리
> - **display**: 동일 토픽에 retry count를 증가시켜 재전송 (`KafkaRetryEvent`) → 상한 초과 시 폐기
> - **marketing**: 실패 레코드를 MongoDB에 저장 (`FailedRecord`) → 수동/배치 재처리

### 일반 Kafka Consumer — 폴링 루프에서 예외 후 재구독

```kotlin
@Component
class BlockingConsumer(
    private val messageBroker: MessageBroker,
    private val applicationEventPublisher: ApplicationEventPublisher,
) {
    @PostConstruct
    fun init() {
        startConsume()
    }

    private fun startConsume() {
        messageBroker.consumeWithBlocking("product-updated") { records ->
            try {
                processRecords(records)
            } catch (e: Exception) {
                log.error("Processing failed, requesting resubscribe", e)
                // 컨슈머 종료 후 재구독 이벤트 발행
                applicationEventPublisher.publishEvent(ResubscribeEvent(this))
                throw e  // 현재 폴링 루프 종료
            }
        }
    }

    @EventListener
    fun onResubscribe(event: ResubscribeEvent) {
        startConsume()
    }
}
```

> 📎 위 Blocking Consumer 패턴도 개념 설명용 예시다. 우리 서비스의 컨슈머는 모두 Reactor Kafka 기반이며, `consumeWithBlocking()` 방식은 사용하지 않는다.

### 재구독 시 주의사항

1. **무한 재시도 방지**: 재구독 횟수에 상한을 두어야 한다. 같은 에러가 반복되면 재구독도 반복되기 때문이다
2. **offset 위치 확인**: 재구독 시 마지막으로 커밋된 offset부터 다시 소비한다. 처리 완료 후 acknowledge하지 못한 메시지는 재처리될 수 있으므로 멱등성을 보장해야 한다
3. **리밸런싱과의 관계**: 재구독 과정에서 LeaveGroup + JoinGroup이 발생하므로, 다른 컨슈머에게도 영향을 줄 수 있다. Static Membership을 사용하면 이 영향을 줄일 수 있다

```
  Dynamic 멤버십 — 재구독 시 다른 컨슈머에 영향
  ──────────────────────────────────────────────
  Consumer A (재구독)           Broker              Consumer B, C
      │                          │                      │
      ├── LeaveGroup ───────────▶│                      │
      │                          ├── "A 이탈"           │
      │                          │                      │
      │                          ├── REBALANCE ────────▶│ ← B, C도 영향!
      │                          │                      │
      │                          │              B: 파티션 반납 ■
      │                          │              C: 파티션 반납 ■
      │                          │              (Stop-the-World)
      │                          │                      │
      ├── JoinGroup ────────────▶│◀── JoinGroup ───────┤
      │                          │   (전체 재배정)       │
      ├── 소비 재개               │          소비 재개 ──┤

  ※ A 하나의 재구독이 그룹 전체 STW를 유발


  Static 멤버십 — 재구독 시 다른 컨슈머에 영향 없음
  ──────────────────────────────────────────────
  Consumer A (재구독)           Broker              Consumer B, C
      │                          │                      │
      │  (LeaveGroup 안 보냄)     │                      │
      │                          │  "A의 instance.id    │
      │                          │   캐싱 중, 대기"      │
      │                          │                      │
      │                          │              B: 소비 계속 ✓
      │                          │              C: 소비 계속 ✓
      │                          │              (영향 없음)
      │                          │                      │
      ├── JoinGroup ────────────▶│                      │
      │   (같은 instance.id)     │                      │
      │                          ├── 캐싱된 할당 반환    │
      │◀── 기존 파티션 그대로 ────┤                      │
      ├── 소비 재개               │          소비 계속 ──┤

  ※ A만 잠시 빠졌다 복귀. B, C는 영향 없음
  ※ 단, session.timeout.ms 이내에 복귀해야 함
```

---

## 핵심 정리

| 개념 | 핵심 |
|------|------|
| 리밸런싱 트리거 | 컨슈머 이탈(timeout), 합류, 파티션 수 변경 |
| session.timeout.ms | **브로커**가 heartbeat 미수신 감지 → 강제 제거 |
| max.poll.interval.ms | **클라이언트**가 poll 간격 초과 감지 → 능동 이탈 |
| assign() | 브로커 관리 없음 → 리밸런싱 없음 → 수동 복구 필요 |
| subscribe() | 브로커 관리 → 자동 재배정 → 리밸런싱 발생 가능 |
| Eager 프로토콜 | 모든 파티션 반납 → 전체 재배정 → **Stop-the-World** |
| Static Membership | 일시 장애 시 LeaveGroup 생략 → **리밸런싱 방지** (Kafka 2.3+) |
| Cooperative Rebalancing | 이동 파티션만 재배정 → **부분 중단** (Kafka 2.4+) |
