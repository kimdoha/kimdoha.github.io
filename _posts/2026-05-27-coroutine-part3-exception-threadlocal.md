---
title: "코루틴 톺아보기 (3) — 예외 처리와 withContext의 Spring Event 손실 패턴"
date: 2026-05-27 10:00:00 +0900
categories: [Kotlin, Coroutine]
tags: [kotlin, coroutine, exception-handling, coroutineexceptionhandler, withcontext, threadlocal, spring-transaction, mdc, runblocking, cancellationexception]
toc: true
---

2편에서 구조화된 동시성과 Job 트리를 통한 취소 전파를 다뤘다. 코루틴이 트리를 이루고, 실패가 전파되고, 취소가 협조적이라는 것까지 이해했다면 — 이제 실전에서 가장 많이 부딪히는 문제를 볼 차례다. 예외가 `launch`와 `async`에서 다르게 전파되는 규칙, `CoroutineExceptionHandler`의 동작 위치, 그리고 `withContext`로 Dispatcher를 전환할 때 Spring의 ThreadLocal 기반 트랜잭션이 유실되는 문제를 다룬다.

> 이 글의 예제 코드는 Kotlin + Spring 기반이며, 핵심 동작에 집중하기 위해 일부 보일러플레이트는 생략했다.

---

## 코루틴 예외 전파 규칙

### launch vs async — 예외가 전파되는 방식이 다르다

코루틴에서 예외가 발생하면 `launch`와 `async`가 전혀 다르게 동작한다. 이 차이를 모르면 예외가 어디서 터지는지 예측할 수 없다.

```kotlin
// launch: 예외가 즉시 부모로 전파된다
scope.launch {
    throw RuntimeException("launch 실패")
    // → 즉시 부모 Job에 예외 전파
    // → 형제 코루틴 취소
    // → CoroutineExceptionHandler에서 처리 (있다면)
}

// async: 예외는 결과 소비자에게는 await()에서 다시 던져진다
// 단, 일반 부모 Job 아래에서는 실패 즉시 부모를 취소한다
val deferred = scope.async {
    throw RuntimeException("async 실패")
    // → Deferred에 저장됨 (await에서 다시 던질 용도)
    // → 동시에 부모 Job에도 실패 전파 (supervisor가 아닌 경우)
}

deferred.await()  // ← 여기서 RuntimeException이 던져진다
```

비교:

| | `launch` | `async` |
|---|---|---|
| 반환 타입 | `Job` | `Deferred<T>` |
| 예외 전파 시점 | 발생 즉시 | 결과 소비자에게는 `await()` 시, 부모에게는 즉시 |
| 예외 전파 대상 | 부모 Job | `await()` 호출자 + 부모 Job (supervisor 제외) |
| 주 용도 | 결과가 필요 없는 작업 (fire-and-forget) | 결과를 기다리는 작업 |

### 그런데 async도 부모에게 전파된다

중요한 뉘앙스가 있다. `async`의 예외가 `await()` 시점까지 보류된다는 것은 **호출자에게 던져지는 시점**에 대한 이야기다. 구조화된 동시성 아래에서 `async`의 실패는 **부모에게도 전파**된다.

```kotlin
suspend fun example() = coroutineScope {
    val deferred = async {
        delay(100)
        throw RuntimeException("async 실패")
    }

    delay(200)
    println("여기까지 도달하는가?")  // 도달하지 않음!
    deferred.await()
}
```

`async`가 100ms에 실패하면, `coroutineScope`에 즉시 전파되어 `coroutineScope` 블록 전체가 취소된다. `await()`를 호출하기 전에 이미 취소된다.

```
0ms    async 시작
100ms  async 실패: RuntimeException
         │
         ├─→ coroutineScope에 전파 (구조화된 동시성)
         │     └─→ coroutineScope 블록 전체 취소
         │
         └─→ deferred에 예외 저장 (await에서 다시 던질 용도)
```

`await()` 없이도 예외가 전파되는 이유: **구조화된 동시성**에서 자식의 실패는 부모에게 보고된다. `await()`는 결과를 받기 위한 수단이지, 예외 전파의 트리거가 아니다.

### supervisorScope에서의 async

`supervisorScope` 안에서는 자식의 실패가 부모에게 전파되지 않으므로, `async`의 예외는 진짜 `await()` 시점에서만 처리된다.

```kotlin
suspend fun example() = supervisorScope {
    val deferred = async {
        delay(100)
        throw RuntimeException("async 실패")
    }

    delay(200)
    println("여기까지 도달한다!")  // 출력됨

    runCatching { deferred.await() }
        .onFailure { println("예외 처리: ${it.message}") }
}
// 출력:
// 여기까지 도달한다!
// 예외 처리: async 실패
```

---

## CoroutineExceptionHandler

### 마지막 안전망

`CoroutineExceptionHandler`는 처리되지 않은 예외를 잡는 **최후의 안전망**이다. Java의 `Thread.uncaughtExceptionHandler`와 유사하다.

```kotlin
val handler = CoroutineExceptionHandler { context, exception ->
    logger.error("코루틴 예외: ${context[CoroutineName]?.name}", exception)
    // 알림 발송, 메트릭 기록 등
}

val scope = CoroutineScope(Dispatchers.Default + handler + CoroutineName("worker"))

scope.launch {
    throw RuntimeException("처리되지 않은 예외")
    // → handler에서 잡힘
}
```

### Handler가 동작하는 위치

`CoroutineExceptionHandler`는 아무 데나 붙이면 동작하는 것이 아니다. **루트 코루틴**(scope에서 직접 시작된 코루틴)에서만 동작한다.

```kotlin
val handler = CoroutineExceptionHandler { _, e ->
    println("잡힘: ${e.message}")
}

val scope = CoroutineScope(Dispatchers.Default + handler)

// ✅ 동작: scope에서 직접 시작한 루트 코루틴
scope.launch {
    throw RuntimeException("루트에서 실패")
}

// ❌ 동작 안 함: 자식 코루틴에 handler를 붙여도 무시된다
scope.launch {
    launch(handler) {
        throw RuntimeException("자식에서 실패")
        // handler 무시 → 부모로 전파 → scope의 handler에서 잡힘
    }
}
```

| 위치 | Handler 동작 |
|---|---|
| Scope 레벨 (루트 코루틴) | 동작 |
| 자식 코루틴 | 무시됨 (부모로 전파 후 루트에서 처리) |
| `coroutineScope` 내부 | 무시됨 (`coroutineScope`가 예외를 다시 던짐) |
| `supervisorScope`의 직계 자식 | 동작 (직계 자식이 루트처럼 취급) |

### async에서는 동작하지 않는다

`CoroutineExceptionHandler`는 `launch`에서만 동작한다. `async`의 예외는 `Deferred`에 캡슐화되어 `await()`에서 던져지므로, Handler가 개입할 시점이 없다.

```kotlin
val handler = CoroutineExceptionHandler { _, e ->
    println("잡힘: ${e.message}")  // 호출되지 않음
}

val scope = CoroutineScope(Dispatchers.Default + handler)

val deferred = scope.async {
    throw RuntimeException("async 실패")
}

runCatching { deferred.await() }
    .onFailure { println("await에서 처리: ${it.message}") }
// 출력: await에서 처리: async 실패
```

---

## 예외 처리 패턴 정리

### 어디서 잡을 것인가

```kotlin
// 패턴 1: try-catch (가장 직접적)
// 주의: catch (Exception)은 CancellationException도 잡으므로 반드시 재던져야 한다
scope.launch {
    try {
        riskyOperation()
    } catch (e: CancellationException) {
        throw e  // 취소는 반드시 다시 던짐
    } catch (e: Exception) {
        logger.error("실패", e)
    }
}

// 패턴 2: runCatching (Kotlin 관용적)
// 주의: runCatching도 CancellationException을 삼킨다 (아래 별도 설명)
scope.launch {
    runCatching { riskyOperation() }
        .onSuccess { result -> process(result) }
        .onFailure { e -> logger.error("실패", e) }
}

// 패턴 3: supervisorScope + 개별 처리
suspend fun loadDashboard() = supervisorScope {
    val a = async { serviceA.getData() }
    val b = async { serviceB.getData() }

    Dashboard(
        dataA = runCatching { a.await() }.getOrDefault(emptyList()),
        dataB = runCatching { b.await() }.getOrDefault(emptyList()),
    )
}

// 패턴 4: CoroutineExceptionHandler (최후의 안전망)
val scope = CoroutineScope(
    Dispatchers.Default + SupervisorJob() + CoroutineExceptionHandler { _, e ->
        logger.error("처리되지 않은 예외", e)
        alertService.notify(e)
    }
)
```

### runCatching과 CancellationException 주의

`runCatching`은 모든 `Throwable`을 잡는다 — `CancellationException`까지도. 이것은 코루틴의 취소 메커니즘을 방해할 수 있다.

```kotlin
// ❌ 위험: CancellationException까지 삼킨다
scope.launch {
    val result = runCatching {
        delay(1000)  // 여기서 취소되면 CancellationException 발생
        fetchData()
    }.getOrDefault(emptyList())
    // 취소됐는데도 계속 실행됨!
    process(result)
}

// ✅ 안전: CancellationException은 다시 던진다
scope.launch {
    val result = try {
        fetchData()
    } catch (e: CancellationException) {
        throw e  // 취소는 다시 던짐
    } catch (e: Exception) {
        logger.error("실패", e)
        emptyList()
    }
    process(result)
}
```

`runCatching`을 코루틴 안에서 사용할 때는 `CancellationException`이 잡히지 않도록 주의해야 한다. Kotlin 표준 라이브러리에서 아직 이에 대한 공식 해결책은 없으므로, suspend 함수에서는 `try-catch`로 `CancellationException`을 명시적으로 재던지거나, `CancellationException`을 걸러내는 확장 함수를 사용하는 것이 안전하다.

```kotlin
// 코루틴 안전한 runCatching 확장
inline fun <T> runSuspendCatching(block: () -> T): Result<T> {
    return try {
        Result.success(block())
    } catch (e: CancellationException) {
        throw e
    } catch (e: Throwable) {
        Result.failure(e)
    }
}
```

---

## withContext와 ThreadLocal의 충돌

### 문제: Spring Event가 사라졌다

Kotlin Coroutine + Spring WebFlux 프로젝트에서 삭제 API를 구현했다. `applicationEventPublisher.publishEvent()`를 호출했지만, `@TransactionalEventListener`가 이벤트를 수신하지 못했다. 예외도 로그도 없었다.

```kotlin
transactionHandler.runInTransaction {
    repository.deleteAll(ids)
}.run {
    applicationEventPublisher.publishEvent(DataChangedEvent(id, relatedId))
}
```

DB 삭제는 정상 수행됐고, `publishEvent()`도 호출됐지만 리스너가 반응하지 않았다.

### 실행 환경

WebFlux는 Netty EventLoop 위에서 non-blocking으로 동작하지만, JOOQ/JPA를 통한 DB 작업은 blocking이다. 이 둘을 연결하기 위해 DB 작업을 전용 스레드 풀(`dbDispatcher`)에서 실행하는 래퍼를 사용한다.

```kotlin
class CoroutineTransactionHandler(
    private val transactionHandler: TransactionHandler,
    private val threadPool: ThreadPoolComponent
) {
    suspend fun <T> runInTransaction(block: () -> T): T {
        return withContext(threadPool.dbDispatcher) {
            transactionHandler.runInTransaction { block() }
        }
    }
}
```

```kotlin
open class TransactionHandler {
    @Transactional(propagation = Propagation.REQUIRED)
    open fun <T> runInTransaction(run: () -> T): T = run()
}
```

실행 구조:

```
Coroutine (Netty EventLoop)
    │
    │  withContext(dbDispatcher)
    ▼
DB Thread Pool ─── @Transactional ─── JOOQ/JPA 실행
    │
    │  withContext 종료
    ▼
Coroutine (이전 CoroutineContext로 재개)  ← 여기서 publishEvent() 호출
```

### 원인: ThreadLocal은 스레드를 넘지 못한다

Spring의 `TransactionSynchronizationManager`는 ThreadLocal 기반으로 동작한다. 하나의 스레드가 하나의 요청을 처리하는 Servlet 모델에서 설계됐기 때문이다. 여기서는 `PlatformTransactionManager` 기반 imperative transaction 기준이다. Reactive transaction은 Reactor Context를 사용하므로 전파 방식이 다르다.

```
Thread-A가 요청을 처리하는 동안의 ThreadLocal 상태:

┌─────────────────────────────────────────────────┐
│  ThreadLocal<Map>                                │
│                                                  │
│  DataSource       → Connection (autoCommit=false)│
│  TransactionStatus → ACTIVE                      │
│  synchronizations → [EventListener1, ...]        │
└─────────────────────────────────────────────────┘
         ↑               ↑              ↑
     Service         Repository    EventPublisher
     (같은 Thread-A에서 접근)
```

`withContext(dbDispatcher)` 안에서 `@Transactional`이 실행되면, DB Thread의 ThreadLocal에 TX 컨텍스트가 바인딩된다. `withContext` 블록이 끝나면 코루틴은 이전 coroutine context로 재개된다(같은 물리 스레드라는 보장은 없다). 이때 DB Thread에 바인딩되어 있던 ThreadLocal 트랜잭션 상태는 재개 스레드로 자동 전달되지 않는다.

```
시간 ────────────────────────────────────────────────────▶

DB Thread-1:
  ┌─ @Transactional 시작 ─────────────────────────┐
  │  ThreadLocal: TX ✅                             │
  │  DELETE 실행 → COMMIT → ThreadLocal 해제         │
  └─────────────────────────────────────────────────┘

                 withContext 종료 → 스레드 전환

Netty Thread-3:
  ┌─ publishEvent() ─────────────────────────────┐
  │  ThreadLocal: TX ❌ (비어있음)                  │
  │  @TransactionalEventListener → TX 없으니 무시   │
  └───────────────────────────────────────────────┘
```

`@TransactionalEventListener`는 `TransactionSynchronizationManager`에서 활성 트랜잭션을 찾고, 없으면 **에러 없이 무시**한다. 이것이 예외도 로그도 없던 이유다.

### 해결: 이벤트를 트랜잭션 안에서 발행한다

`@EventListener`를 사용하는 경우, 이벤트 데이터를 TX 전에 준비하고 TX 결과에 의존하지 않으면 된다.

```kotlin
suspend fun delete(targetIds: List<Int>) {
    if (targetIds.isEmpty()) return

    // 1. 삭제 전에 필요한 데이터 조회
    val items = transactionHandler.runReadOnlyTransaction {
        repository.findByIds(targetIds)
    }

    // 2. 삭제 실행
    transactionHandler.runInTransaction {
        repository.deleteAll(targetIds)
    }

    // 3. 미리 준비한 데이터로 이벤트 발행
    items.forEach {
        applicationEventPublisher.publishEvent(DataChangedEvent(it.id, it.relatedId))
    }
}
```

`@TransactionalEventListener`가 필요한 경우, 이벤트 발행을 트랜잭션 블록 안으로 옮긴다.

```kotlin
// 이벤트 데이터는 TX 전에 준비
val events = items.map { DataChangedEvent(it.id, it.relatedId) }

// TX 안에서 발행 → DB Thread + TX 활성 상태
transactionHandler.runInTransaction {
    repository.deleteAll(targetIds)

    events.forEach { applicationEventPublisher.publishEvent(it) }
}
// @TransactionalEventListener(AFTER_COMMIT) 정상 실행
```

| 방식 | @EventListener | @TransactionalEventListener | 롤백 시 이벤트 |
|------|:-:|:-:|:-:|
| TX 밖에서 발행 | 동작 | 무시됨 | 발행 안 됨 (예외 전파로 미도달) |
| TX 안에서 발행 | 동작 | 동작 | 자동 폐기 |

---

## Spring + 코루틴 통합 시 추가 주의점

### MDC (Mapped Diagnostic Context) 유실

SLF4J의 MDC도 ThreadLocal 기반이다. `withContext`로 스레드가 전환되면 MDC 컨텍스트가 유지된다는 보장이 없다.

```kotlin
// ❌ withContext 이후 MDC 값이 보장되지 않는다
MDC.put("requestId", "req-123")
withContext(Dispatchers.IO) {
    logger.info("처리 시작")  // MDC.get("requestId") → null일 수 있음
}
```

해결: `kotlinx-coroutines-slf4j` 모듈의 `MDCContext`를 사용한다.

```kotlin
import kotlinx.coroutines.slf4j.MDCContext

// ✅ MDCContext를 Context에 추가하면 스레드 전환 시 MDC가 복사된다
MDC.put("requestId", "req-123")
withContext(Dispatchers.IO + MDCContext()) {
    logger.info("처리 시작")  // MDC.get("requestId") → "req-123"
}
```

`MDCContext`는 `ThreadContextElement`를 구현한다. `MDCContext()` 생성 시점의 MDC 상태를 캡처하여, `withContext`로 스레드가 전환될 때 새 스레드에 복사하고, 블록이 끝나면 원래대로 복원한다.

주의: `MDCContext`를 사용하더라도, 코루틴 내부에서 `MDC.put()`으로 변경한 값은 다음 suspension point에서 유실될 수 있다. `MDCContext`가 복원하는 것은 생성 시점에 캡처한 스냅샷이기 때문이다. 코루틴 안에서 MDC를 변경해야 한다면 `withContext(MDCContext())`를 다시 감싸서 새 스냅샷을 만들어야 한다.

```
┌─ MDCContext 동작 흐름 ──────────────────────────────┐
│                                                      │
│  Thread-A (MDC: requestId=req-123)                   │
│    │                                                 │
│    │  withContext(IO + MDCContext())                  │
│    │    ① Thread-A의 MDC 저장                         │
│    │    ② Thread-B에 MDC 복사                         │
│    ▼                                                 │
│  Thread-B (MDC: requestId=req-123)  ← 복사됨         │
│    │  작업 실행                                       │
│    │                                                 │
│    │  블록 종료                                       │
│    │    ③ Thread-B의 MDC 원래대로 복원                 │
│    ▼                                                 │
│  Thread-A (MDC: requestId=req-123)  ← 원래대로       │
└──────────────────────────────────────────────────────┘
```

### runBlocking을 서버에서 쓰면 안 되는 이유

`runBlocking`은 현재 스레드를 **blocking**하면서 코루틴을 실행한다. 코루틴의 핵심인 "스레드를 점유하지 않는다"를 정면으로 거스른다.

```kotlin
// ❌ WebFlux + runBlocking: Netty EventLoop 스레드가 blocking된다
@GetMapping("/products/{id}")
fun getProduct(@PathVariable id: Long): Product {
    return runBlocking {
        // Netty EventLoop 스레드가 여기서 멈춤
        // → 다른 요청도 이 EventLoop에서 처리되어야 하는데 대기
        // → 처리량 급감, 최악의 경우 전체 서버 응답 불가
        productService.getProduct(id)
    }
}
```

WebFlux의 Netty EventLoop는 소수의 스레드(보통 CPU 코어 수)로 모든 요청을 처리한다. `runBlocking`으로 하나를 막으면 해당 EventLoop에 바인딩된 모든 연결이 대기한다.

Spring MVC(Servlet 모델)에서도 `runBlocking`은 스레드 풀의 스레드를 점유하므로, 높은 동시 요청 환경에서는 스레드 풀 고갈 위험이 있다.

```kotlin
// ✅ WebFlux: suspend 함수를 직접 사용
@GetMapping("/products/{id}")
suspend fun getProduct(@PathVariable id: Long): Product {
    return productService.getProduct(id)
}

// ✅ Spring MVC: 코루틴이 필요 없으면 쓰지 않는다
// 필요하면 Controller에서 suspend fun을 사용하고 Spring이 처리하도록 위임
```

---

## 정리

| 개념 | 핵심 |
|---|---|
| launch 예외 | 발생 즉시 부모에게 전파. CoroutineExceptionHandler에서 처리 가능 |
| async 예외 | Deferred에 저장. await()에서 던져짐. 단, 구조화된 동시성에서 부모에게도 전파됨 |
| CoroutineExceptionHandler | 루트 코루틴의 launch에서만 동작하는 최후의 안전망 |
| runCatching + 코루틴 | CancellationException까지 잡을 수 있어 위험. 재던지거나 확장 함수 사용 |
| withContext + ThreadLocal | 스레드가 전환되면 ThreadLocal 유실. Spring TX, MDC 모두 해당 |
| @TransactionalEventListener | TX 없는 스레드에서 호출하면 에러 없이 무시 |
| MDCContext | ThreadContextElement로 MDC를 코루틴 컨텍스트에 포함시켜 전환 시 복사 |
| runBlocking | 현재 스레드를 blocking. 서버 환경에서 사용 금지 |

코루틴과 Spring을 함께 쓸 때 가장 자주 빠지는 함정은 **ThreadLocal 유실**이다. Spring의 트랜잭션, 보안 컨텍스트, MDC 등은 모두 ThreadLocal 기반인데, `withContext`는 스레드를 전환한다. 이 둘이 만나면 조용히 기능이 사라지고, 에러도 로그도 남지 않는다. "어떤 스레드에서 실행되는가"를 항상 의식하는 것이 코루틴 + Spring 환경에서 가장 중요한 습관인 것 같다.