---
title: "코루틴 톺아보기 (2) — CoroutineScope와 Context: 구조화된 동시성이 버그를 막는 법"
date: 2026-05-26 10:00:00 +0900
categories: [Kotlin, Coroutine]
tags: [kotlin, coroutine, coroutinescope, coroutinecontext, structured-concurrency, supervisorscope, globalscope, async, cancellation]
toc: true
---

1편에서 suspend 함수가 CPS 변환과 Continuation으로 동작하는 원리를 다뤘다. 코루틴 하나의 동작은 이해했지만, 실제 서비스에서는 여러 코루틴이 동시에 실행된다. 이때 "누가 이 코루틴들을 관리하는가", "하나가 실패하면 나머지는 어떻게 되는가"를 결정하는 것이 `CoroutineScope`와 `CoroutineContext`다.

> 이 글의 예제 코드는 Kotlin 기반이며, 핵심 동작에 집중하기 위해 일부 보일러플레이트는 생략했다.

---

## CoroutineContext — 코루틴의 메타데이터

### Context란 무엇인가

모든 코루틴은 `CoroutineContext`를 갖는다. 이것은 코루틴이 실행되는 데 필요한 메타데이터의 집합이다. 단일 값이 아니라 **Key-Value 쌍의 불변 컬렉션**이며, `Map`과 유사하게 동작한다.

```
┌─────────────── CoroutineContext ───────────────┐
│                                                 │
│  ┌─────────────┐  ┌──────────────────────────┐  │
│  │ Job         │  │ CoroutineDispatcher      │  │
│  │ (생명주기)   │  │ (실행 스레드)             │  │
│  └─────────────┘  └──────────────────────────┘  │
│                                                 │
│  ┌─────────────┐  ┌──────────────────────────┐  │
│  │ CoroutineName│  │ CoroutineExceptionHandler│  │
│  │ (디버깅용)   │  │ (최상위 예외 처리)        │  │
│  └─────────────┘  └──────────────────────────┘  │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │ ThreadContextElement (MDC, ThreadLocal)  │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

각 요소(Element)는 고유한 `Key`를 갖는다. 하나의 Context 안에 같은 Key를 가진 Element는 하나만 존재한다.

### Context 합성 — `+` 연산자

두 Context를 `+`로 합치면 새로운 Context가 만들어진다. **같은 Key가 있으면 오른쪽이 이긴다.**

```kotlin
val ctx1 = Dispatchers.Default + CoroutineName("worker")
// ctx1: {Dispatcher=Default, CoroutineName=worker}

val ctx2 = ctx1 + Dispatchers.IO
// ctx2: {Dispatcher=IO, CoroutineName=worker}
// → Dispatcher가 Default에서 IO로 교체됨

val ctx3 = ctx2 + Job()
// ctx3: {Dispatcher=IO, CoroutineName=worker, Job=새 Job}
```

합성 규칙은 단순하다:
- Key가 다르면 **병합**
- Key가 같으면 **덮어쓰기** (오른쪽 우선)

### Context에서 Element 꺼내기

```kotlin
val context = Dispatchers.IO + CoroutineName("fetcher") + Job()

// Key로 접근
val dispatcher = context[ContinuationInterceptor] as? CoroutineDispatcher  // Dispatchers.IO
val name = context[CoroutineName]               // CoroutineName("fetcher")
val job = context[Job]                          // Job 인스턴스

// 없는 Key는 null
val handler = context[CoroutineExceptionHandler] // null
```

`CoroutineDispatcher`는 자체 `Key` companion object가 없고, `AbstractCoroutineContextElement(ContinuationInterceptor)`를 상속한다. 따라서 context에서 꺼낼 때는 `ContinuationInterceptor` Key를 사용해야 한다. `Job`이나 `CoroutineName`은 자체 `Key`가 있어 직접 접근 가능하다.

---

## CoroutineScope — Context를 담는 그릇

### Scope의 정의

`CoroutineScope`는 놀라울 정도로 단순한 인터페이스다:

```kotlin
public interface CoroutineScope {
    public val coroutineContext: CoroutineContext
}
```

멤버가 `coroutineContext` 하나뿐이다. Scope는 Context를 담는 그릇이며, 이 Scope 안에서 `launch`나 `async`로 새 코루틴을 시작하면 **Scope의 Context를 상속**받는다.

```kotlin
val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO + CoroutineName("parent"))

scope.launch {
    // 이 코루틴의 Context:
    // Dispatcher = IO (상속)
    // CoroutineName = parent (상속)
    // Job = 새 Job (자동 생성, scope의 SupervisorJob이 부모가 됨)
}
```

`CoroutineScope()` 생성 시 context에 Job이 없으면 기본 `Job()`이 자동 추가된다. 위 예제처럼 명시적으로 `SupervisorJob()`을 넣으면 자식 실패가 다른 자식에게 전파되지 않는 스코프가 된다.

### coroutineScope vs CoroutineScope()

이름이 같아서 혼동하기 쉽지만 전혀 다르다.

| 항목 | `coroutineScope { }` (소문자, 함수) | `CoroutineScope()` (대문자, 생성자) |
|---|---|---|
| 역할 | 현재 코루틴의 하위 스코프를 만듦 | 새 스코프를 생성 |
| 부모-자식 | 호출한 코루틴의 lexical child | 호출한 suspend 함수의 lexical child가 아님. 생명주기를 직접 관리해야 한다 |
| 취소 전파 | 자식 실패 → 부모에게 전파 | 전달된 Job을 스코프의 생명주기로 사용. 구조화된 관계가 필요하면 `coroutineScope {}`를 우선 사용하고, 독립 스코프는 소유 객체의 생명주기에 맞춰 직접 cancel()해야 한다 |
| suspend | suspend 함수 (일시 중단 가능) | 일반 함수 |
| 완료 조건 | 모든 자식이 끝나야 반환 | 명시적으로 cancel() 호출 필요 |

```kotlin
// coroutineScope — 구조화된 동시성. 안전하다.
suspend fun fetchProductData(productNo: Long): ProductData = coroutineScope {
    val product = async { productRepository.findById(productNo) }
    val reviews = async { reviewRepository.findByProductNo(productNo) }
    ProductData(product.await(), reviews.await())
    // 둘 다 끝나야 반환. 하나가 실패하면 나머지도 취소.
}

// CoroutineScope() — 비구조화. 생명주기를 직접 관리해야 한다.
class ProductService {
    private val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())

    fun fireAndForget(event: Event) {
        scope.launch {
            // 부모 코루틴과 무관하게 실행
            eventPublisher.publish(event)
        }
    }

    fun destroy() {
        scope.cancel()  // 명시적으로 정리해야 한다
    }
}
```

### GlobalScope는 왜 위험한가

`GlobalScope`는 `Job`이 없는 빈 `CoroutineContext`를 가진 스코프다. 부모 Job이 없으므로 구조화된 동시성 바깥에서 동작하며, 어디서든 코루틴을 시작할 수 있지만 자동으로 취소되지 않는다.

```kotlin
// 위험: 누가 이 코루틴을 취소하는가?
GlobalScope.launch {
    while (true) {
        delay(1000)
        logger.info("heartbeat")
    }
}
// → 애플리케이션이 살아있는 동안 취소되지 않고 계속 실행될 수 있음
// → 테스트에서 제어 불가
// → 예외 발생 시 아무도 모름
```

문제:
1. **생명주기 관리 불가** — 취소할 수단이 없다 (반환된 Job을 변수에 담지 않으면)
2. **구조화된 동시성 이탈** — 부모-자식 관계가 없어 취소 전파가 안 된다
3. **리소스 누수** — 논리적 범위(요청, 컴포넌트 생명주기 등)가 끝나도 코루틴이 취소되지 않고 계속 실행될 수 있다
4. **테스트 불가** — Scope를 교체할 수 없어 테스트에서 실행 순서를 제어할 수 없다

대안: `CoroutineScope()`로 명시적인 스코프를 만들거나, 프레임워크가 제공하는 스코프(Spring의 `CoroutineScope` Bean 등)를 사용한다.

---

## 구조화된 동시성 (Structured Concurrency)

### 코루틴은 트리를 이룬다

코루틴은 독립적으로 존재하지 않는다. `launch`나 `async`로 새 코루틴을 시작하면, 부모 코루틴의 `Job`을 부모로 갖는 자식 `Job`이 만들어진다. 이것이 **Job 계층 구조**다.

```
CoroutineScope
  └─ Job (scope의 Job)
       ├─ Job (launch A)
       │    ├─ Job (launch A-1)
       │    └─ Job (launch A-2)
       └─ Job (launch B)
            └─ Job (async B-1)
```

이 트리 구조가 **구조화된 동시성(Structured Concurrency)** 의 핵심이다. 규칙은 세 가지:

### 규칙 1: 부모는 모든 자식이 완료될 때까지 기다린다

```kotlin
suspend fun process() = coroutineScope {
    launch {
        delay(1000)
        println("자식 1 완료")
    }
    launch {
        delay(2000)
        println("자식 2 완료")
    }
    println("coroutineScope 블록 끝")
    // 여기서 반환되지 않는다 — 자식 2가 끝날 때까지 대기
}
// 출력:
// coroutineScope 블록 끝
// 자식 1 완료
// 자식 2 완료
// (이 시점에서 process()가 반환됨)
```

### 규칙 2: 부모가 취소되면 모든 자식도 취소된다

```kotlin
val parentJob = scope.launch {
    launch {
        delay(Long.MAX_VALUE)
        println("자식 1 — 출력 안 됨")
    }
    launch {
        delay(Long.MAX_VALUE)
        println("자식 2 — 출력 안 됨")
    }
}

delay(100)
parentJob.cancel()  // 부모 취소 → 자식 전부 취소
```

취소가 전파되는 흐름:

```
parentJob.cancel()
  │
  ├─→ 자식 1의 Job.cancel()
  │     └─→ CancellationException 발생
  │          └─→ delay()에서 즉시 중단
  │
  └─→ 자식 2의 Job.cancel()
        └─→ CancellationException 발생
             └─→ delay()에서 즉시 중단
```

### 규칙 3: 자식이 실패하면 부모도 실패하고, 형제도 취소된다

이것이 가장 중요한 규칙이다.

```kotlin
suspend fun fetchAll() = coroutineScope {
    val product = async {
        delay(100)
        throw RuntimeException("DB 연결 실패")  // 자식 1 실패
    }
    val reviews = async {
        delay(5000)  // 5초 걸리는 작업
        println("출력 안 됨")
    }
    product.await()  // 여기서 예외 발생
    reviews.await()
}
```

실패 전파 흐름:

```
시간 ────────────────────────────────────────▶

0ms    자식 1 (product) 시작
       자식 2 (reviews) 시작
100ms  자식 1 실패: RuntimeException
         │
         ├─→ 부모 coroutineScope에 예외 전파
         │     │
         │     └─→ 자식 2 (reviews) 취소
         │           └─→ CancellationException
         │
         └─→ coroutineScope 블록에서 RuntimeException 던짐
```

자식 2는 5초가 필요하지만, 자식 1이 100ms 만에 실패하면 즉시 취소된다. **불필요한 작업이 계속 실행되는 것을 방지**한다.

> **async와 예외 전파**: `async`의 예외는 `await()`에서 관찰되지만, child 실패 자체는 구조화된 동시성에 의해 부모를 취소한다. 즉 `await()`를 호출하지 않더라도 자식의 실패는 부모에게 전파된다.

### 취소의 협조적 특성

코루틴 취소는 **협조적(cooperative)** 이다. `cancel()`을 호출해도 코루틴이 즉시 멈추는 것이 아니다. 코루틴이 스스로 취소 상태를 확인해야 한다.

```kotlin
// ❌ 취소되지 않는 코루틴
scope.launch {
    var count = 0
    while (count < 1_000_000) {
        // CPU 집약 작업: suspend point가 없어서 취소를 확인할 기회가 없다
        count++
    }
}

// ✅ 취소에 협조하는 코루틴 — 방법 1: ensureActive()
scope.launch {
    var count = 0
    while (count < 1_000_000) {
        ensureActive()  // 취소 상태면 CancellationException 던짐
        count++
    }
}

// ✅ 취소에 협조하는 코루틴 — 방법 2: isActive 확인
scope.launch {
    var count = 0
    while (isActive && count < 1_000_000) {
        count++
    }
}

// ✅ 취소에 협조하는 코루틴 — 방법 3: yield()
scope.launch {
    var count = 0
    while (count < 1_000_000) {
        yield()  // 다른 코루틴에게 실행 기회를 양보하면서 취소도 확인
        count++
    }
}
```

`delay()`, `withContext()`, `yield()` 같은 suspend 함수들은 내부에서 취소 상태를 확인한다. CPU 집약 루프에서는 직접 `ensureActive()` 또는 `isActive`를 사용해야 한다.

### CancellationException은 특별하다

코루틴에서 `CancellationException`은 **정상적인 취소**로 취급된다. 다른 예외와 달리 부모를 실패시키지 않는다.

```kotlin
suspend fun example() = coroutineScope {
    val job1 = launch {
        delay(1000)
        println("job1 완료")
    }
    val job2 = launch {
        delay(500)
        job1.cancel()  // job1을 취소해도
        println("job2 완료")
    }
    // job1이 CancellationException으로 취소되어도
    // 부모(coroutineScope)와 job2에는 영향 없음
}
// 출력:
// job2 완료
```

| 예외 타입 | 부모 전파 | 형제 취소 | 의미 |
|---|---|---|---|
| `CancellationException` | 부모를 실패시키지 않음 | 취소 안 됨 | 정상적인 취소 |
| 그 외 예외 (`RuntimeException` 등) | 전파됨 | 취소됨 | 실패 |

---

## supervisorScope — 부분 실패 허용

### 왜 필요한가

`coroutineScope`는 자식 하나가 실패하면 나머지를 모두 취소한다. 이것은 "전부 성공하거나 전부 실패"하는 시나리오에 적합하다. 하지만 "일부는 실패해도 나머지는 계속 진행해야 하는" 경우도 있다.

```kotlin
// coroutineScope: 상품 조회 실패 → 리뷰 조회도 취소 (원하지 않는 동작)
suspend fun loadDashboard() = coroutineScope {
    val products = async { productService.getTop10() }        // 실패!
    val reviews = async { reviewService.getRecent() }         // 취소됨
    val notifications = async { notificationService.get() }   // 취소됨
    // → 대시보드 전체가 빈 화면
}
```

### supervisorScope의 동작

`supervisorScope`는 내부적으로 `SupervisorJob`을 사용하여 **자식 간 실패 전파를 막는다**. 단, block 자체의 예외는 scope를 실패시키며, `launch`의 uncaught exception은 `CoroutineExceptionHandler` 등으로 별도 처리해야 한다.

```kotlin
suspend fun loadDashboard() = supervisorScope {
    val products = async { productService.getTop10() }        // 실패!
    val reviews = async { reviewService.getRecent() }         // 계속 실행
    val notifications = async { notificationService.get() }   // 계속 실행

    // 실패한 것만 개별 처리
    val productResult = runCatching { products.await() }.getOrDefault(emptyList())
    val reviewResult = runCatching { reviews.await() }.getOrDefault(emptyList())
    val notificationResult = runCatching { notifications.await() }.getOrDefault(emptyList())

    Dashboard(productResult, reviewResult, notificationResult)
}
```

실패 전파 비교:

```
coroutineScope:
  자식 1 실패 ──→ 부모 실패 ──→ 자식 2 취소
                              ──→ 자식 3 취소

supervisorScope:
  자식 1 실패 ──→ 자식 1만 실패
                  자식 2 계속 실행
                  자식 3 계속 실행
```

### SupervisorJob 주의사항

`SupervisorJob`을 직접 사용할 때 흔한 실수:

```kotlin
// ❌ 잘못된 사용: launch 안에서 SupervisorJob을 쓰면 부모-자식 관계가 끊어진다
scope.launch(SupervisorJob()) {
    // 이 launch의 Job이 SupervisorJob의 자식이 되지만,
    // scope.launch가 만드는 Job과는 관계가 없어진다
    // → 구조화된 동시성이 깨짐
}

// ✅ 올바른 사용: supervisorScope를 쓴다
scope.launch {
    supervisorScope {
        launch { /* 실패해도 형제에 영향 없음 */ }
        launch { /* 독립적으로 실행 */ }
    }
}
```

---

## 실전 패턴

### 패턴 1: coroutineScope + async로 병렬 호출

서비스 레이어에서 여러 도메인을 동시에 조회하는 패턴이다.

```kotlin
suspend fun toExcelData(info: ProductInquiryInfo): List<ExcelData> {
    val parentNos = info.inquiries.map { it.inquiryNo }.toSet()
    val productNos = info.inquiries.map { it.mallProductNo.toLong() }.toSet()

    val (replies, malls, partners, globalProducts) = coroutineScope {
        Tuples.of(
            async { inquiryRepository.getReplies(parentNos) },
            async { mallService.getMalls(info.mallNos) },
            async { partnerService.getPartners(info.partnerNos) },
            async { productService.getGlobalProducts(productNos) },
        )
    }.run {
        Tuples.of(t1.await(), t2.await(), t3.await(), t4.await())
    }

    // 4개 조회가 병렬로 실행됨
    // 하나라도 실패하면 나머지도 취소 → 일관된 실패 처리
}
```

`coroutineScope`를 사용하므로:
- 4개 async가 동시에 실행된다
- 하나가 실패하면 나머지도 취소된다 (불필요한 I/O 방지)
- 모든 async가 완료되어야 `coroutineScope` 블록이 반환된다

> **`await()` 위치에 주의**: 병렬성이 깨지는 경우는 `async { }.await()`를 **즉시 연속 호출**할 때다 — 즉 다음 `async`를 시작하기 전에 앞의 결과를 기다리면 순차 실행이 된다. 위 예제처럼 모든 `async`를 먼저 시작한 뒤 나중에 `await()`를 호출하면 병렬로 실행된다. 또한 `coroutineScope`는 블록 끝에서 모든 자식 코루틴이 완료될 때까지 대기하므로, `.run { }` 안의 `await()`는 이미 완료된 결과를 즉시 반환할 뿐 병렬성에 영향을 주지 않는다.

### 패턴 2: supervisorScope로 부분 실패 허용

대시보드, 알림 집계 등 일부 실패가 전체를 망가뜨리면 안 되는 경우:

```kotlin
suspend fun getProductSummary(productNo: Long): ProductSummary = supervisorScope {
    val product = async { productService.getProduct(productNo) }
    val reviewCount = async { reviewService.countByProductNo(productNo) }
    val relatedProducts = async { recommendService.getRelated(productNo) }

    ProductSummary(
        product = product.await(),  // 필수: 실패하면 전체 실패
        reviewCount = runCatching { reviewCount.await() }.getOrDefault(0),
        relatedProducts = runCatching { relatedProducts.await() }.getOrDefault(emptyList()),
    )
}
```

### 패턴 3: withTimeout으로 타임아웃 제어

```kotlin
suspend fun getProductWithTimeout(productNo: Long): Product? {
    return withTimeoutOrNull(3000L) {
        // 3초 이내에 완료되지 않으면 null 반환
        productService.getProduct(productNo)
    }
}
```

`withTimeout`은 시간 초과 시 `TimeoutCancellationException`(CancellationException의 하위 클래스)을 던진다. `withTimeoutOrNull`은 예외 대신 null을 반환한다.

---

## 정리

| 개념 | 핵심 |
|---|---|
| CoroutineContext | Key-Value 쌍의 불변 컬렉션. Job, Dispatcher, Name 등 코루틴 메타데이터를 담는다 |
| `+` 연산자 | 두 Context를 합성. 같은 Key면 오른쪽이 덮어쓴다 |
| CoroutineScope | Context를 담는 그릇. 이 안에서 시작된 코루틴은 Context를 상속받는다 |
| coroutineScope | 현재 코루틴의 하위 스코프. 모든 자식 완료를 기다리고, 실패를 전파한다 |
| GlobalScope | 부모 없는 최상위 스코프. 생명주기 관리 불가. 사용 자제 |
| 구조화된 동시성 | Job 트리 구조. 부모 취소 → 자식 취소, 자식 실패 → 부모 실패 |
| CancellationException | 정상적인 취소. 부모에게 전파되지 않는다 |
| supervisorScope | 자식 실패가 형제에게 전파되지 않는다. 부분 실패 허용 |

구조화된 동시성의 가치는 단순하다. 코루틴이 **트리 구조**로 관리되기 때문에, 하나가 실패했을 때 나머지를 어떻게 할지 — 전부 취소할지(coroutineScope), 나머지는 계속할지(supervisorScope) — 를 명시적으로 선택할 수 있다. GlobalScope처럼 트리 밖에서 코루틴을 시작하면 이 보호막이 사라진다.

다음 편에서는 코루틴에서 **예외가 전파되는 규칙**과, `withContext`로 Dispatcher를 전환할 때 **ThreadLocal이 유실되는 문제** — Spring의 `@TransactionalEventListener`가 동작하지 않는 실제 사례를 다뤄보겠다.