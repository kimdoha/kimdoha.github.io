---
title: "Redis 분산락과 청크 처리 — 실무 패턴 정리"
date: 2026-05-19 10:00:00 +0900
categories: [JVM, Spring]
tags: [oom, concurrency, redis, distributed-lock, redisson, lettuce, chunking, batch]
---

> **TL;DR**: OOM은 메모리 누수만으로 발생하지 않는다. 동시성 제어 실패로 데이터가 꼬이고, 대량 데이터를 한 번에 처리하려다 힙이 터진다. 실무에서 겪은 동시성 제어와 대량 데이터 처리의 교훈을 정리한다.

---

## 1. 재고 차감과 동시성 제어

커머스에서 재고 차감은 동시성 이슈의 대표적인 사례다. 여러 사용자가 동시에 같은 상품을 주문하면 재고가 음수가 되거나, 초과 판매가 발생할 수 있다.

### 방법 1: SQL 단일 문장으로 원자적 처리

```sql
UPDATE pd_stock
SET stock_cnt = stock_cnt - #{orderCnt}
WHERE stock_no = #{stockNo}
  AND stock_cnt >= #{orderCnt}   -- 재고 부족 시 affected rows = 0
```

InnoDB는 UPDATE 실행 시 해당 행에 배타적 잠금(exclusive row lock)을 건다. 같은 행에 대한 동시 UPDATE는 잠금 대기 후 순차 실행되므로, **조건 확인과 값 변경이 하나의 문장 안에서 원자적으로** 이루어진다. WHERE 조건을 통과하지 못하면 `affected rows = 0`이 반환되고, 이를 재고 부족으로 판단하면 된다.

**주의**: 이 패턴은 **단일 SQL 문장** 안에서만 안전하다. 아래처럼 SELECT와 UPDATE를 분리하면 동시성 문제가 생긴다.

```kotlin
// 위험한 패턴: SELECT → 비즈니스 로직 → UPDATE
val stock = stockRepository.findByStockNo(stockNo)  // snapshot 읽기
if (stock.stockCnt >= orderCnt) {                    // 이 시점에 다른 트랜잭션이 이미 차감했을 수 있음
    stockRepository.decreaseStock(stockNo, orderCnt) // lost update 가능
}
```

이 경우 `SELECT ... FOR UPDATE`로 비관적 잠금을 걸거나, WHERE 조건에 재고 체크를 포함한 단일 UPDATE 문을 사용해야 한다.

> **근거**: MySQL 공식 문서 — InnoDB는 `UPDATE` 문에 대해 대상 행에 exclusive lock을 설정한다. REPEATABLE READ에서 일반 `SELECT`는 MVCC 스냅샷을 읽지만, `UPDATE`의 WHERE 절은 현재 커밋된 최신 값을 기준으로 평가된다(current read). ([MySQL - InnoDB Locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locks-set.html))

### 방법 2: Redis 기반 분산락

DB 잠금만으로 부족하거나, 여러 서비스에 걸친 리소스를 보호해야 할 때 Redis 분산락을 사용한다.

#### INCR 패턴 (단순하지만 위험)

```kotlin
val count = redisTemplate.opsForValue().increment(lockKey)
if (count == 1L) {
    // 락 획득 성공
    try {
        doSomething()
    } finally {
        redisTemplate.delete(lockKey)
    }
}
```

INCR은 Redis의 단일 스레드 특성상 원자적이다. 결과가 1이면 최초 호출자(=락 획득자)로 판단할 수 있다.

**문제**: INCR 후 EXPIRE로 TTL을 설정하는 두 단계가 별개 명령이다. **INCR은 성공했는데 EXPIRE 호출 전에 애플리케이션이 죽으면 TTL 없는 키가 영구히 남는다.** 이 키가 남으면 이후 모든 요청이 락 획득에 실패한다. Redis 공식 문서도 이 race condition을 명시하고 있다.

> *"if the client crashes just after the INCR and before the EXPIRE the key will be leaked"* — [Redis INCR Documentation](https://redis.io/docs/latest/commands/incr/)

#### SET NX PX 패턴 (권장)

```kotlin
// 단일 명령으로 락 획득 + TTL 설정
val acquired = redisTemplate.opsForValue()
    .setIfAbsent(lockKey, requestId, Duration.ofSeconds(10))

if (acquired == true) {
    try {
        doSomething()
    } finally {
        // Lua 스크립트로 본인 락만 해제 (다른 클라이언트의 락을 삭제하지 않도록)
        val script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
        """.trimIndent()
        redisTemplate.execute(RedisScript.of(script, Long::class.java), listOf(lockKey), requestId)
    }
}
```

`SET key value NX PX milliseconds`는 키가 없을 때만 설정하면서 TTL까지 한 번에 거는 원자적 명령이다. Redis 공식 분산락 문서가 권장하는 패턴이며, Lua 스크립트로 본인의 락만 해제하는 것이 핵심이다.

#### 용도에 따른 해제 전략

`SET NX PX`로 키를 잡는 것까지는 동일하지만, **해제 방식은 용도에 따라 다르다.**

| 용도 | 값 | 해제 방식 | 이유 |
|------|---|----------|------|
| **분산 뮤텍스** (공유 자원 보호) | `UUID` | Lua 스크립트 (GET+비교+DEL) | TTL 만료 후 다른 클라이언트 락 삭제 방지 |
| **중복 요청 차단** (멱등성 키) | 고정 값 | plain `DELETE` 또는 TTL 자연 만료 | 소유권 충돌이 구조적으로 없음 |

**분산 뮤텍스**: 재고 차감처럼 공유 자원을 보호할 때는, 작업이 TTL보다 오래 걸리면 다른 클라이언트가 락을 획득할 수 있다. 이때 원래 클라이언트의 `finally`가 새 클라이언트의 락을 삭제하면 안 된다. UUID로 소유자를 식별하고 Lua로 본인 것만 삭제해야 한다.

```kotlin
// 분산 뮤텍스: UUID + Lua 해제
val lockValue = UUID.randomUUID().toString()
val acquired = redisTemplate.opsForValue()
    .setIfAbsent(lockKey, lockValue, Duration.ofSeconds(10))

if (acquired == true) {
    try {
        doSomething()
    } finally {
        val script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
        """.trimIndent()
        redisTemplate.execute(RedisScript.of(script, Long::class.java), listOf(lockKey), lockValue)
    }
}
```

**중복 요청 차단**: 같은 요청이 동시에 두 번 들어오는 것을 막는 용도에서는, 요청 A와 요청 B가 **같은 사용자의 같은 동작**이다. 소유권 구분이 필요 없으므로 plain `DELETE`로 충분하다. 실무에서는 이 패턴을 **임계 구간 핸들러**로 감싸서 사용하면 깔끔하다.

```kotlin
// CriticalSectionHandler: 중복 요청 차단 (멱등성 키)
class CriticalSectionHandler(
    private val reactiveRedisTemplate: ReactiveStringRedisTemplate,
) {
    suspend fun <T> runInCriticalSection(
        key: String,
        expireTime: Duration,
        block: suspend () -> T,
    ): T? {
        val acquired = reactiveRedisTemplate.opsForValue()
            .setIfAbsentAndAwait(key, "1", expireTime)

        if (!acquired) return null  // 중복 요청 → 진입 차단

        return try {
            block()
        } finally {
            runCatching { reactiveRedisTemplate.deleteAndAwait(key) }
        }
    }
}
```

`setIfAbsent` + TTL을 한 번에 설정하고, `try-finally`로 해제를 보장한다. 값이 고정(`"1"`)이고 plain `DELETE`를 쓰지만, 중복 요청 차단 용도에서는 소유권 충돌이 구조적으로 발생하지 않으므로 안전하다.

위 코드에서 사용하는 `ReactiveStringRedisTemplate`은 **Spring Data Redis**가 제공하는 리액티브 Redis 클라이언트다. 내부적으로 **Lettuce**를 커넥션 드라이버로 사용하며, Netty 기반 비동기 I/O로 동작한다. Spring Boot에서 `spring-boot-starter-data-redis-reactive` 의존성만 추가하면 별도 설정 없이 사용할 수 있다. 대부분의 Spring 기반 프로젝트에서 Redis를 다루는 표준적인 방식이다.

> **근거**: Redis 공식 문서 — 분산락 구현에 `SET resource_name my_random_value NX PX 30000`을 사용하고, 해제는 GET+비교+DEL을 Lua로 원자적으로 수행하라고 권장한다. ([Redis - Distributed Locks](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)) / Spring Data Redis 공식 문서 — Lettuce는 Spring Boot의 기본 Redis 드라이버이며, 리액티브 지원을 포함한다. ([Spring Data Redis](https://docs.spring.io/spring-data/redis/reference/redis/drivers.html))

#### Lettuce vs Redisson

위 코드들은 모두 Spring Data Redis + **Lettuce** 조합이다. Redisson은 별도 라이브러리인데, 둘의 역할이 다르다.

| 구분 | Lettuce | Redisson |
|------|---------|----------|
| **역할** | Redis 명령을 보내는 **드라이버** | Redis 위에 자바 자료구조/동기화를 제공하는 **프레임워크** |
| **수준** | 저수준 — `GET`, `SET`, `DEL` 직접 호출 | 고수준 — `RLock`, `RMap`, `RQueue` 등 추상화 |
| **분산락** | 직접 구현 (`SET NX PX` + Lua) | 내장 (`getLock()` + Watchdog 자동 TTL 갱신) |
| **Spring 통합** | Spring Boot 기본 드라이버 | 별도 의존성 추가 필요 |

Lettuce로는 `SET NX PX` + Lua 해제를 직접 작성하고, Redisson은 `lock.lock()` 한 줄이면 내부적으로 SET NX + Watchdog + Lua 해제를 알아서 처리한다. 중복 요청 차단 수준이면 Lettuce + `CriticalSectionHandler`로 충분하고, 재진입 락이나 Watchdog이 필요한 분산 뮤텍스라면 Redisson이 선택지가 된다.

#### 분산 뮤텍스가 필요하다면: Redisson

```kotlin
val lock = redissonClient.getLock("lock:stock:${stockNo}")
try {
    // leaseTime을 지정하지 않으면 Watchdog이 활성화되어 TTL 자동 갱신
    lock.lock()
    doSomething()
} finally {
    if (lock.isHeldByCurrentThread) {
        lock.unlock()
    }
}
```

`SET NX PX` + Lua 패턴으로 직접 구현하는 것도 가능하지만, TTL 관리, 재진입, 공정성 등을 직접 다루기는 까다롭다. Redisson의 `RLock`은 내부적으로 **Watchdog** 메커니즘을 사용한다. 락을 보유한 스레드가 살아있는 동안 TTL을 자동으로 갱신(기본 30초)하므로, 수동 TTL 관리의 race condition이 없다.

**주의**: `tryLock(waitTime, leaseTime, unit)`처럼 **leaseTime을 명시하면 Watchdog은 비활성화**된다. 지정한 시간이 지나면 작업 완료 여부와 관계없이 락이 만료된다. Watchdog 자동 갱신을 원하면 `lock()`이나 `tryLock(waitTime, unit)`처럼 leaseTime을 생략해야 한다.

**RedLock에 대해**: 여러 독립 Redis master에 대해 과반수 합의로 락을 거는 Redlock 알고리즘은 Redisson에서 `RedissonRedLock`으로 제공되었으나 **deprecated** 되었다. 최신 문서에서는 단일 노드 `RLock` 또는 `RFencedLock` 사용을 권장한다. 대부분의 커머스 재고 처리처럼 짧은 임계 구간에는 RLock으로 충분하다.

> **근거**: Redisson 공식 문서 — Watchdog은 `leaseTime` 파라미터가 지정되지 않은 경우에만 활성화되며, 기본 갱신 주기는 `lockWatchdogTimeout / 3`이다. `RedissonRedLock`은 deprecated 되었다. ([Redisson - Distributed Locks](https://redisson.pro/docs/data-and-services/locks-and-synchronizers/))

---

## 2. 대량 데이터는 반드시 청크로 처리한다

평소에는 수십 건이던 상품 동기화 배치가, 대량 등록이나 이벤트 시 수천~수만 건으로 급증할 수 있다. 이 데이터를 한 번에 Feign 호출하거나 IN 쿼리에 넘기면 **타임아웃, 메모리 초과, 쿼리 플래너 비효율**이 발생한다.

### 청크 처리 패턴

```kotlin
val allProducts = fetchProducts()  // 수천 건

allProducts.chunked(CHUNK_SIZE).forEach { chunk ->
    // Feign 호출, IN 쿼리, 벌크 INSERT 등
    processProducts(chunk)
}
```

### Chunked + Semaphore: 동시성까지 제한하기

청크로 쪼개더라도 모든 청크를 동시에 비동기 호출하면 외부 서비스나 DB 커넥션 풀에 부하가 몰린다. **Semaphore로 동시 실행 수를 제한**하면 메모리와 외부 시스템 부하를 동시에 방어할 수 있다.

```kotlin
val semaphore = Semaphore(MAX_CONCURRENT_CALLS)  // 최대 5개 동시 호출

stockNos.distinct().chunked(120).map { chunk ->
    async {
        semaphore.withPermit {
            stockClient.getStocks(chunk)
        }
    }
}.awaitAll().flatten()
```

`chunked(120)`으로 요청 크기를 제한하고, `Semaphore(5)`로 동시 호출 수를 제한한다. 이렇게 하면 1,200건의 재고 조회가 120건씩 10개 청크로 나뉘고, 최대 5개만 동시에 실행된다. 외부 API 호출, Feign 클라이언트, WebClient 호출에 모두 적용할 수 있는 패턴이다.

### Double-Chunking: 읽기와 쓰기의 청크를 분리하기

Spring Batch 같은 구조에서는 **읽기 청크**와 **쓰기 청크**를 다르게 가져가는 것이 효과적이다. 읽기는 크게 가져오되, 실제 DB 삭제/수정은 더 작은 단위로 처리한다.

```kotlin
// Spring Batch: chunk=2000으로 읽음
// Writer 내부에서는 200건씩 재분할하여 삭제
override fun write(items: Chunk<ReviewInfo>) {
    val reviewNos = items.map { it.reviewNo }

    // 첨부파일 삭제: 200건씩
    reviewNos.chunked(BATCH_SIZE).forEach { chunk ->
        deleteAttachments(chunk)
    }

    // 리뷰 삭제: 200건씩
    reviewNos.chunked(BATCH_SIZE).forEach { chunk ->
        deleteReviews(chunk)
    }
}
```

읽기 청크(2,000건)가 크면 DB round-trip을 줄일 수 있고, 쓰기 청크(200건)가 작으면 트랜잭션 크기와 락 범위를 제한할 수 있다. 두 값을 독립적으로 튜닝할 수 있는 것이 장점이다.

### 청크 사이즈 출발점 예시

| 대상 | 출발점 사이즈 | 이유 |
|------|-----------|------|
| DB IN 쿼리 | 100~500 | MySQL 쿼리 플래너 효율, 패킷 크기 제한 |
| Feign/REST 호출 | 50~200 | 요청 페이로드 크기, 타임아웃 |
| 벌크 INSERT/UPDATE | 500~1,000 | 트랜잭션 크기, undo log |

> 위 수치는 일반적으로 사용되는 범위를 예시로 들었고, 실제 최적값은 네트워크 지연, 데이터 크기, DB/서비스 부하에 따라 다르다. 부하 테스트로 시스템별 튜닝이 필요하다.

### 실패 격리도 중요하다

청크 처리만으로는 부족하다. 하나의 상품 처리가 실패했을 때 나머지 상품까지 막히면 안 된다.

```kotlin
// 나쁜 예: 하나 실패하면 전체 중단
allProducts.chunked(CHUNK_SIZE).forEach { chunk ->
    processProducts(chunk)  // 하나라도 예외 → 전체 실패
}

// 좋은 예: 실패 격리
allProducts.chunked(CHUNK_SIZE).forEach { chunk ->
    chunk.forEach { product ->
        runCatching { processProduct(product) }
            .onFailure { e ->
                log.error("상품 처리 실패: ${product.productNo}", e)
                retryQueue.add(product)  // 재시도 큐로 이동
            }
    }
}
```

**원칙**: 개별 건의 처리 실패가 전체를 막지 않도록 **실패 범위를 격리**한다. `runCatching`으로 건별 실패를 잡고, 실패한 건만 재시도 큐로 보내면 나머지는 정상 처리된다.

---

## 3. 정리

| 영역 | 체크 항목 |
|------|----------|
| **동시성** | 재고 차감은 단일 SQL 문으로 처리하거나 분산락 사용. SELECT → UPDATE 분리 금지 |
| **분산락** | `SET NX PX` + Lua 해제 또는 Redisson 사용. INCR 단독 사용 금지 |
| **대량 처리** | Feign 호출, IN 쿼리는 반드시 chunked |
| **실패 격리** | 개별 건 실패가 전체를 막지 않도록 `runCatching`으로 격리 |
| **모니터링** | 배치/게이트웨이 등 간과하기 쉬운 시스템에도 메모리 알람 필수 |

---

## 참고 자료

- [MySQL 8.4 Reference Manual - InnoDB Locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locks-set.html)
- [Redis - Distributed Locks](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
- [Redis INCR Command](https://redis.io/docs/latest/commands/incr/)
- [Redisson - Distributed Locks and Synchronizers](https://redisson.pro/docs/data-and-services/locks-and-synchronizers/)
- [Spring Data Redis - Drivers](https://docs.spring.io/spring-data/redis/reference/redis/drivers.html)
