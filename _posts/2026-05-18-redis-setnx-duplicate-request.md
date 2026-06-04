---
title: "Redis SET NX로 중복 요청 차단하기 — 락이 아닌 Best-Effort 방어선의 한계와 활용"
date: 2026-05-18 10:00:00 +0900
categories: [Redis, Backend]
tags: [redis, distributed-lock, setnx, concurrency, duplicate-request, backend]
toc: true
---

> **TL;DR**: Redis의 `SET NX PX`는 동일 리소스에 대한 중복 요청을 빠르게 차단하는 데 유용하다. 다만 단일 Redis 인스턴스 기반 락은 페일오버, TTL 초과, unlock 방식에 따라 상호 배제가 깨질 수 있다. 이 글은 "완전한 분산 락" 구현법이 아니라, 상품 수정 API에서 **best-effort 방어선**으로 Redis 락을 사용한 사례와 한계를 정리한다.

## 배경

상품 수정 API에서 동일 상품에 대한 동시 요청이 들어오면, 중복 INSERT 자체는 DB의 UNIQUE KEY + `onDuplicateKeyIgnore()`로 방어할 수 있다. 하지만 UK가 방어하지 못하는 문제들이 존재한다:

- **Lost Update**: 추가 요청과 삭제 요청이 동시에 실행되면, 호출자는 "성공" 응답을 받지만 최종 상태가 기대와 다를 수 있다 (추가했는데 삭제됨, 삭제했는데 다시 생김)
- **이벤트 중복 발행**: 동시 요청이 각각 history 적재, webhook 발송, 색인 요청을 발행하여 외부 시스템에 중복 처리를 유발한다
- **TOCTOU (Time-of-check to Time-of-use)**: 검증 시점의 상태와 실행 시점의 상태가 달라 무효화된 항목이 INSERT될 수 있다

이를 줄이기 위해 Redis의 `SET NX` (SETNX) 명령을 활용한 **단일 키 락 패턴**으로, **동일 상품에 대한 중복 요청 자체를 진입 시점에서 차단(best-effort)** 한다.

정확히 말하면 이 방식은 결제, 재고 차감, 포인트 적립처럼 강한 정합성이 필요한 도메인에 그대로 적용할 수 있는 "안전한 분산 락"이 아니다. 중복 요청 자체가 드물고, 실패해도 DB 제약과 트랜잭션이 일부 피해를 줄여줄 수 있는 상황에서 선택한 **best-effort 중복 요청 차단 장치**에 가깝다.

---

## Redis 기반 락 핵심 개념

### SET NX란?

Redis의 `SET key value NX EX seconds` 명령은 **키가 존재하지 않을 때만** 값을 설정한다.

```
SET resource_name my_random_value NX PX 30000
```

| 옵션 | 의미 |
|------|------|
| `NX` | 키가 없을 때만 설정 (Not eXists) |
| `PX 30000` | 만료 시간 30초 (밀리초 단위) |
| `my_random_value` | 이 락을 식별하는 고유 값 |

이 명령 하나로 **"키 존재 여부 확인 + 키 설정 + TTL 설정"이 원자적(atomic)으로** 실행된다.

> Redis 공식 문서: [Distributed Locks with Redis](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)

### 안전 속성 (Safety & Liveness)

Redis 공식 문서에서 정의하는 분산 락의 핵심 속성:

| 속성 | 설명 | 단일 인스턴스 | Redlock (다중 인스턴스) |
|------|------|:------------:|:---------------------:|
| **상호 배제 (Mutual Exclusion)** | 동일 시점에 하나의 클라이언트만 락을 보유 | ⚠️ 조건부 | ✅ |
| **데드락 방지 (Deadlock-free)** | 락을 보유한 클라이언트가 crash 되어도 TTL에 의해 자동 해제 | ✅ | ✅ |
| **내결함성 (Fault-tolerant)** | Redis 노드 일부가 장애여도 락 획득/해제 가능 | ❌ | ✅ |

> 이 글의 구현은 **단일 Redis 인스턴스 기반 락**을 사용한다. 상호 배제는 **Redis 정상 동작 + TTL 내 완료 + Master 페일오버 없음** 조건에서만 기대할 수 있다. 이 조건이 깨지면 두 클라이언트가 동시에 실행될 수 있으며, 이에 대한 대응은 [Graceful Degradation](#redis-장애-시-graceful-degradation)과 [단일 Redis 인스턴스의 한계](#단일-redis-인스턴스의-한계와-허용-근거) 섹션에서 다룬다.

---

## 동작 흐름

### 시퀀스 다이어그램

```
Client A                    Redis                     Client B
   │                          │                          │
   │  SETNX "LOCK:P:10001"    │                          │
   │─────────────────────────>│                          │
   │        true (획득)        │                          │
   │<─────────────────────────│                          │
   │                          │                          │
   │ 비즈니스 로직 실행 중...      │  SETNX "LOCK:P:10001"    │
   │                          │<─────────────────────────│
   │                          │       false (거부)       │
   │                          │─────────────────────────>│
   │                          │                          │
   │                          │              PRODUCT_MODIFY_DUPLICATION
   │                          │                    예외 반환
   │                          │                          │
   │  로직 완료                 │                          │
   │  DEL "LOCK:P:10001"      │                          │
   │─────────────────────────>│                          │
   │        해제 완료           │                          │
   │<─────────────────────────│                          │
```

### 상태 흐름도

```
                    ┌─────────────────┐
                    │  API 요청 수신  │
                    └────────┬────────┘
                             │
                    ┌────────v────────┐
                    │  Redis SETNX    │
                    │  (NX + TTL)     │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
         ┌────v────┐    ┌────v────┐    ┌────v────┐
         │  true   │    │  false  │    │ 장애    │
         │ (획득)  │    │ (실패)  │    │ (연결X) │
         └────┬────┘    └────┬────┘    └────┬────┘
              │              │              │
    ┌─────────v─────────┐    │    ┌─────────v─────────┐
    │ 비즈니스 로직     │    │    │ 락 없이 실행      │
    │ 실행              │    │    │ (Graceful         │
    │                   │    │    │  Degradation)     │
    └─────────┬─────────┘    │    └─────────┬─────────┘
              │              │              │
    ┌─────────v─────────┐    │              │
    │  finally:         │    │              │
    │  Redis DEL        │    │              │
    └─────────┬─────────┘    │              │
              │              │              │
         ┌────v────┐   ┌─────v─────────┐    │
         │  응답   │   │ 중복 요청 예외│    │
         │  반환   │   │ 반환          │    │
         └─────────┘   └───────────────┘    │
                                       ┌────v────┐
                                       │  응답   │
                                       │  반환   │
                                       └─────────┘
```

---

## 구현 코드

### 1. Lock 정보 클래스

```kotlin
// CsInfo.kt — 락 키 생성 규칙 정의
abstract class CsInfo(
    val key: Any,                              // 락 대상 식별자
    val type: CriticalSectionType,             // 락 유형
    val expiredTm: Duration = Duration.ofMillis(10000L)  // TTL
) {
    fun getRedisKey(prefix: String) = "$prefix:${type.name}:$key"
    // 예: "CS:PRODUCT_MODIFY:10000001"
}

// ProductModifyCsInfo.kt — 상품 수정 전용
class ProductModifyCsInfo(key: Any)
    : CsInfo(key, CriticalSectionType.PRODUCT_MODIFY, Duration.ofSeconds(10L))
```

Redis 키 구조:
```
CS:PRODUCT_MODIFY:{productNo}
│  │               │
│  │               └─ 락 대상 (상품번호)
│  └─ 락 유형
└─ prefix
```

### 2. Lock 획득/해제 핸들러

```kotlin
// CriticalSectionHandler.kt
@Component
class CriticalSectionHandler(
    private val reactiveRedisTemplate: ReactiveRedisTemplate<String, Any>,
) {
    private val redisKeyPrefix = RedisKeys.CRITICAL_SECTION.value

    suspend fun <R> runInSuspendCriticalSection(csInfo: CsInfo, block: suspend () -> R): R {
        val redisKey = csInfo.getRedisKey(redisKeyPrefix)

        // 1. Lock 획득 시도 (SETNX + TTL, 원자적 실행)
        val success = try {
            reactiveRedisTemplate.opsForValue()
                .setIfAbsentAndAwait(redisKey, 1, csInfo.expiredTm)  // 값 1 = placeholder
        } catch (e: RedisConnectionFailureException) {
            // Redis 장애 시: 락 없이 실행 (서비스 가용성 우선)
            return block()
        }

        // 2. Lock 획득 성공 → 로직 실행 + finally에서 해제
        if (success) {
            return try {
                block()
            } finally {
                reactiveRedisTemplate.deleteAndAwait(redisKey)
            }
        }

        // 3. Lock 획득 실패 → 중복 요청 차단
        throw NCPException(
            ProductErrorCode.PRODUCT_MODIFY_DUPLICATION,
            arrayOf(csInfo.key.toString())
        )
    }
}
```

> ⚠️ **`finally { deleteAndAwait }` 주의점**: `block()`이 성공한 후 `deleteAndAwait`에서 예외가 발생하면, 비즈니스 성공 결과가 Redis 삭제 예외로 덮어씌워질 수 있다. SETNX 시점에 Redis가 정상이었으므로 DEL 시점 장애 확률은 낮지만, 프로덕션에서는 `finally` 블록 내 DEL 실패를 catch하여 로그만 남기거나 TTL 만료에 의존하는 방식도 고려할 수 있다.

### 값 `1`의 의미 — 정석 패턴과의 차이

`setIfAbsentAndAwait(redisKey, 1, ...)` 에서 `1`은 **의미 없는 placeholder**다. 락의 동작은 키의 존재 여부로만 판단하기 때문에 값 자체는 중요하지 않다.

Redis 공식 문서의 정석 패턴은 값으로 **랜덤 값(UUID 등)** 을 사용한다:

```
SET resource_name my_random_value NX PX 30000
```

이유는 **락 해제 시 소유자 검증**을 위해서다:

| | 정석 패턴 | 우리 구현 |
|--|-----------|-----------|
| 값 | UUID (랜덤) | `1` (고정) |
| 해제 시 | 값 비교 후 DEL (Lua script) | 무조건 DEL |
| 목적 | 다른 클라이언트의 락을 실수로 해제하지 않음 | 단순화 |

우리 코드에서 소유자 검증을 생략한 이유와 그 리스크:
- 락 획득과 해제가 **동일 요청의 `try-finally` 안에서** 항상 쌍으로 실행된다
- 대부분의 요청이 TTL(10초) 내에 완료되므로, 정상 상황에서는 다른 클라이언트의 락을 삭제할 가능성이 낮다

> ⚠️ **리스크**: GC pause, DB 커넥션 풀 고갈, 외부 API 지연 등으로 비즈니스 로직이 TTL(10초)을 초과하면, 원래 클라이언트의 락이 만료된 후 다른 클라이언트가 획득한 락을 `finally` 블록에서 삭제할 수 있다. 이는 상호 배제가 깨지는 시나리오다. 정석대로라면 UUID + Lua script 기반 compare-and-delete로 방지해야 하지만, 우리는 이 락이 best-effort 수준임을 전제로 단순성을 택했다.

### 3. 서비스 레이어 적용

```kotlin
@Service
class CustomPropertyCommandService(
    private val cs: CriticalSectionHandler,
    // ...
) {
    suspend fun addCustomProperties(
        productNo: Long, mallNo: Int, partnerNo: Int, propValueNos: Set<Int>
    ): CustomPropertyPartialResult {
        // productNo 단위로 Lock 획득 후 실행
        return cs.runInSuspendCriticalSection(ProductModifyCsInfo(productNo)) {
            updateCustomPropertyMappings(productNo, mallNo, partnerNo, propValueNos) {
                customPropertyRepository.insertPropMappings(mallNo, productNo, it)
            }
        }
    }
}
```

---

## 동시 요청 시나리오

### Case 1: 동일 상품 동시 수정 — Lock이 차단

```
시간  요청A (추가항목 추가)              요청B (추가항목 삭제)
──────────────────────────────────────────────────────────
T0    SETNX "CS:...:10001" → true
T1    검증 + INSERT 실행 중...
T2                                       SETNX "CS:...:10001" → false
T3                                       → PRODUCT_MODIFY_DUPLICATION 예외
T4    COMMIT + DEL key
T5                                       (재시도 시 정상 처리)
```

### Case 2: 서로 다른 상품 — 간섭 없음

```
시간  요청A (상품 10001)                 요청B (상품 10002)
──────────────────────────────────────────────────────────
T0    SETNX "CS:...:10001" → true
T1                                       SETNX "CS:...:10002" → true
T2    병렬 실행 (서로 다른 키)           병렬 실행
T3    COMMIT + DEL                       COMMIT + DEL
```

### Case 3: 추가항목 API와 상품 수정 API 동시 요청 — 상호 배제

```
시간  추가항목 추가 API                  상품 전체 수정 API
──────────────────────────────────────────────────────────
T0    SETNX "CS:PRODUCT_MODIFY:10001" → true
T1    추가항목 INSERT 중...
T2                                       SETNX "CS:PRODUCT_MODIFY:10001" → false
T3                                       → PRODUCT_MODIFY_DUPLICATION 예외
T4    COMMIT + DEL key
```

> 동일한 `PRODUCT_MODIFY` 타입의 Redis 키를 공유하므로, 정상 상황에서는 상품 수정과 추가항목 변경이 동시에 들어와도 상호 배제된다.

---

## 설계 결정과 트레이드오프

### 왜 Non-blocking (대기 없이 즉시 실패) 방식인가?

```
┌───────────────────────────────────────────────────────────────┐
│              Blocking Lock              Non-blocking Lock      │
│                                                               │
│  요청B ──> [대기...대기...대기] ──>    요청B ──> [즉시 실패]  │
│            실행                                  재시도       │
│                                                               │
│  장점: 요청 유실 없음                 장점: 응답 빠름         │
│  단점: 대기 시간 불확실              단점: 클라이언트 재시도  │
│        커넥션 점유                          필요              │
└───────────────────────────────────────────────────────────────┘
```

우리는 **Non-blocking 방식**을 채택했다:
- API 서버가 WebFlux(비동기) 기반이므로 스레드 블로킹은 부적절
- 중복 요청은 클라이언트 UI에서의 더블클릭 등 비정상 케이스이므로 즉시 에러 반환이 적합
- 락 대기 시 커넥션 풀 고갈 위험 제거

### 왜 TTL이 필요한가?

```
정상: 요청 → Lock 획득 → 로직 실행 → finally에서 DEL → Lock 해제
비정상: 요청 → Lock 획득 → 서버 crash (DEL 실행 안 됨)
                               └─ TTL 10초 후 자동 만료 → 데드락 방지
```

### Redis 장애 시 Graceful Degradation

```kotlin
val success = try {
    reactiveRedisTemplate.opsForValue()
        .setIfAbsentAndAwait(redisKey, 1, csInfo.expiredTm)
} catch (e: RedisConnectionFailureException) {
    // Redis가 죽어도 API는 동작한다 (락 없이 실행)
    log.warn("Redis Connection Fail. The logic executed normally, key[${csInfo.key}]")
    return block()
}
```

**주의: Redis 장애 시 중복 요청 차단은 동작하지 않는다.** 락 없이 비즈니스 로직이 그대로 실행되므로, 동시 요청이 들어오면 Lost Update나 이벤트 중복 발행이 발생할 수 있다.

| 상황 | 동작 | 중복 방지 |
|------|------|:---------:|
| Redis 정상 | SETNX로 락 획득/거부 | O |
| Redis 장애 | 락 없이 비즈니스 로직 실행 | **X** |

그럼에도 이렇게 설계한 이유:

- **Redis 장애는 일시적이고 드문 케이스**다. 동시 요청 자체도 비정상 상황이므로, 두 조건이 동시에 겹칠 확률은 극히 낮다
- **락 실패 시 API를 죽이면 Redis 장애가 서비스 장애로 확대**된다. 단일 인프라 장애가 전체 상품 수정 API를 중단시키는 것은 과도한 결합
- **락 없이 실행해도 UK + `onDuplicateKeyIgnore()`가 DB 레벨에서 중복 INSERT는 방어**한다. 다만 UK가 방어하지 못하는 Lost Update나 이벤트 중복 발행은 Redis 장애 시 허용된다
- 즉, **중복 요청 차단은 포기하되, 중복 INSERT라는 가장 빈번한 실패는 DB가 방어**하는 구조다

---

## 단일 Redis 인스턴스의 한계와 허용 근거

Redis 공식 문서에서 언급하는 단일 인스턴스 기반 락의 한계:

```
Client A ──> Master (Lock 획득)
                │
                │ crash (복제 전)
                │
             Replica ──> 승격 (Lock 정보 없음)
                │
Client B ──> New Master (Lock 획득) ← 안전 속성 위반!
```

Master-Replica 구조에서 Master가 crash되면 복제 전의 Lock 정보가 유실되어, 두 클라이언트가 동시에 락을 보유하는 상황이 이론적으로 가능하다.

우리는 이 한계를 인지하면서도 단일 인스턴스 방식을 사용한다:
- **중복 요청 차단 목적**이다. 엄격한 상호 배제가 아닌 "최선의 노력(best-effort)" 수준이면 충분하다
- 최악의 경우(락 유실)에도 **UK + `onDuplicateKeyIgnore()`가 DB 레벨에서 중복 INSERT를 방어**한다
- Redlock 알고리즘(Redis 노드 과반수 합의)은 이 유스케이스에 비해 과도한 복잡도를 도입한다

> 참고: 재고 차감처럼 정확한 상호 배제가 필요한 도메인에서는 Redis 락보다 `UPDATE ... SET stock_cnt = stock_cnt - amount WHERE stock_cnt >= amount` 같은 DB 원자 연산이 더 적합할 수 있다. 동시성 제어 방식은 도메인 특성에 따라 달라진다.

---

## 언제 이 방식을 쓰면 안 되는가

이 패턴은 동일 리소스에 대한 중복 요청을 줄이는 **보조 방어선**으로는 유용하지만, 모든 동시성 문제의 정답은 아니다. 다음처럼 한 번의 중복 실행이 금전 손실이나 강한 정합성 위반으로 이어지는 도메인에서는 이 방식만으로 부족하다:

- 결제 승인/취소
- 재고 차감
- 쿠폰 발급
- 포인트 적립/차감
- 계좌성 잔액 변경

이런 경우에는 Redis 락을 최종 방어선으로 두기보다, 문제의 성격에 맞는 더 강한 방식을 우선 검토해야 한다.

| 문제 | 우선 검토할 방식 |
|------|-----------------|
| 같은 요청이 여러 번 들어옴 | Idempotency Key |
| 수량/잔액을 원자적으로 변경 | DB 조건부 UPDATE |
| 읽고 판단한 뒤 변경 | Optimistic Lock / Pessimistic Lock |
| 외부 이벤트 중복 발행 | Outbox Pattern |
| 여러 인스턴스의 캐시 재생성 경쟁 | Request Coalescing / Cache Stampede 방지 |

즉, Redis `SET NX PX`는 "동일 키 요청을 빠르게 한 번만 통과시키는 입구 제어"에는 잘 맞지만, 데이터 정합성의 최종 책임을 맡기기에는 부족하다.

---

## 방어 계층 요약

```
┌─────────────────────────────────────────────┐
│  Layer 1: Redis Lock (중복 요청 차단)       │ ← 동시 요청 자체를 차단
├─────────────────────────────────────────────┤
│  Layer 2: Transaction (데이터 원자성)       │ ← 검증 + 변경을 하나의 트랜잭션
├─────────────────────────────────────────────┤
│  Layer 3: UNIQUE KEY (DB 레벨 제약)         │ ← 중복 INSERT 방어 (Lost Update 미방어)
├─────────────────────────────────────────────┤
│  Layer 4: onDuplicateKeyIgnore (앱 레벨)    │ ← UK 위반 시 에러 대신 무시
└─────────────────────────────────────────────┘
```

각 계층은 독립적으로 동작하며, 상위 계층이 실패해도 하위 계층이 부분적으로 보완한다. 단, Layer 3(UK)은 중복 INSERT만 방어하며, Lost Update나 이벤트 중복 발행은 방어하지 못한다.

---

## References

### Redis 분산 락
- [Redis 공식 문서 — Distributed Locks with Redis](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
- [Redis 공식 문서 — SETNX Command](https://redis.io/docs/latest/commands/setnx/)
- [Redis Lock Glossary](https://redis.io/glossary/redis-lock/)
- [The Twelve Redis Locking Patterns Every Distributed Systems Engineer Should Know](https://medium.com/@navidbarsalari/the-twelve-redis-locking-patterns-every-distributed-systems-engineer-should-know-06f16dfe7375)
- [10 Hidden Pitfalls of Using Redis Distributed Locks](https://leapcell.medium.com/10-hidden-pitfalls-of-using-redis-distributed-locks-b5234ddd6349)

### 동시성 문제 (배경 섹션 관련)
- [Lost Update — Wikipedia](https://en.wikipedia.org/wiki/Write%E2%80%93write_conflict) — Write-write conflict (Lost Update) 정의
- [TOCTOU — CWE-367](https://cwe.mitre.org/data/definitions/367.html) — Time-of-check Time-of-use Race Condition (MITRE CWE 공식 정의)
- [TOCTOU — Wikipedia](https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use) — TOCTOU 개념 설명
