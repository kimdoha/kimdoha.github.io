---
title: "코루틴 톺아보기 (1) — suspend 함수의 스레드 반납 동작 원리"
date: 2026-05-21 10:00:00 +0900
categories: [Kotlin, Coroutine]
tags: [kotlin, coroutine, suspend, cps, continuation, dispatcher, withcontext, limitedparallelism]
toc: true
---

코루틴은 "가벼운 스레드"라고 흔히 소개된다. 하지만 이 한 줄짜리 설명은 오해를 낳기 쉽다. 코루틴은 스레드가 아니며, 스레드 위에서 실행되는 **일시 중단 가능한 연산(suspendable computation)** 이다. 스레드를 점유하지 않으면서도 동기 코드처럼 읽히는 비동기 처리 — 이것이 가능한 이유를 컴파일러 레벨부터 파헤쳐본다.

> 이 글의 예제 코드는 Kotlin 기반이며, 핵심 동작에 집중하기 위해 일부 보일러플레이트는 생략했다.

---

## 왜 코루틴인가 — 스레드 모델의 한계

### 스레드의 비용

많은 JVM 환경에서 스레드 하나의 기본 스택 크기는 대략 1MB 수준이다(`-Xss`로 조정 가능하며, OS와 컨테이너 설정에 따라 달라진다). 동시 요청이 1,000개면 1GB, 10,000개면 10GB다. 물리적으로 스레드를 무한히 만들 수 없기 때문에 스레드 풀을 사용하지만, 풀 크기는 곧 동시 처리 한계를 의미한다.

```
요청 10,000개 동시 처리 시

스레드 모델:
┌─────────────────────────────────────────────────────────┐
│  Thread Pool (200개)                                     │
│  ┌──┐┌──┐┌──┐┌──┐┌──┐ ... ┌──┐                         │
│  │T1││T2││T3││T4││T5│     │200│  ← 200개만 동시 실행     │
│  └──┘└──┘└──┘└──┘└──┘     └──┘                          │
│                                                          │
│  나머지 9,800개 요청 → 큐에서 대기 (blocking)             │
│  메모리: 200 × 1MB = 200MB (스택만)                      │
└─────────────────────────────────────────────────────────┘
* 여기서 "큐"는 ThreadPoolExecutor의 work queue(BlockingQueue<Runnable>)를 가리킨다.
  Tomcat 기준으로 클라이언트 연결은 NIO Acceptor가 수락(max-connections: 8192)하고,
  처리할 스레드가 없으면 executor의 TaskQueue에서 대기한다.

코루틴 모델:
┌─────────────────────────────────────────────────────────┐
│  스레드 소수 (CPU 코어 수) + 코루틴 10,000개              │
│  ┌──┐┌──┐┌──┐┌──┐                                       │
│  │T1││T2││T3││T4│  ← 4개 스레드가 10,000개 코루틴 번갈아 실행 │
│  └──┘└──┘└──┘└──┘                                       │
│                                                          │
│  I/O 대기 시 스레드 반납 → 다른 코루틴 실행               │
│  메모리: 코루틴당 수백 Byte ~ 수 KB                       │
└─────────────────────────────────────────────────────────┘
```

### Blocking의 본질적 문제

전통적인 스레드 모델에서 I/O 호출(DB 쿼리, HTTP 요청)은 응답이 올 때까지 스레드를 **점유한 채 대기**한다. 스레드는 아무 일도 하지 않으면서 1MB의 메모리를 차지하고, 다른 요청은 이 스레드가 풀릴 때까지 큐에서 기다린다.

```kotlin
// 블로킹 모델: 스레드가 DB 응답을 기다리는 동안 아무 일도 못 함
fun getProduct(productNo: Long): Product {
    val product = productRepository.findById(productNo)  // 스레드 blocking (50ms)
    val reviews = reviewRepository.findByProductNo(productNo)  // 스레드 blocking (30ms)
    return product.copy(reviews = reviews)
    // 총 80ms 동안 스레드 1개 독점
}
```

### 콜백 지옥 vs 코루틴

비동기로 해결하려 하면 콜백 지옥에 빠진다. Reactor/RxJava 같은 리액티브 스트림은 콜백을 체이닝으로 풀었지만, 코드가 선형적으로 읽히지 않는다.

```kotlin
// 리액티브 체이닝 — 동작하지만 읽기 어렵다
fun getProduct(productNo: Long): Mono<Product> {
    return productRepository.findById(productNo)
        .flatMap { product ->
            reviewRepository.findByProductNo(productNo)
                .collectList()
                .map { reviews -> product.copy(reviews = reviews) }
        }
}

// 코루틴 — 동기 코드처럼 읽히지만 비동기로 동작한다
suspend fun getProduct(productNo: Long): Product {
    val product = productRepository.findById(productNo)     // suspend point (non-blocking API라면 스레드 반납)
    val reviews = reviewRepository.findByProductNo(productNo) // suspend point
    return product.copy(reviews = reviews)
}
```

코루틴의 `suspend` 함수는 **동기 코드의 가독성**과 **비동기 실행의 효율성**을 동시에 제공한다. 단, `suspend` 키워드 자체가 스레드를 반납하는 것은 아니다. 실제로 suspension이 발생하고, 내부 구현이 non-blocking일 때만 스레드를 점유하지 않는다. 예를 들어 suspend 함수 안에서 blocking JDBC를 호출하면 스레드는 그대로 막힌다. 문제는 "어떻게 non-blocking suspension이 동작하는가?"다.

---

## suspend 함수의 내부 — CPS 변환

### suspend 키워드가 하는 일

`suspend`는 Kotlin 컴파일러에게 보내는 신호다. "이 함수는 중간에 실행을 멈출 수 있다"는 표시이며, 컴파일러는 이 함수를 **CPS(Continuation-Passing Style)** 로 변환한다.

개발자가 작성한 코드:

```kotlin
suspend fun fetchUser(userId: Long): User {
    val profile = fetchProfile(userId)    // suspend point 1
    val orders = fetchOrders(userId)      // suspend point 2
    return User(profile, orders)
}
```

컴파일러가 변환한 코드 (개념적):

```kotlin
fun fetchUser(userId: Long, continuation: Continuation<User>): Any? {
    // Continuation = "중단된 지점부터 다시 시작하기 위한 콜백"
    val sm = continuation as? FetchUserSM ?: FetchUserSM(continuation)

    when (sm.label) {
        0 -> {
            sm.label = 1
            val result = fetchProfile(userId, sm)  // suspend point 1
            if (result == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
            sm.profile = result as Profile
        }
        1 -> {
            sm.profile = sm.result as Profile
            sm.label = 2
            val result = fetchOrders(userId, sm)   // suspend point 2
            if (result == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
            sm.orders = result as List<Order>
        }
        2 -> {
            sm.orders = sm.result as List<Order>
            return User(sm.profile, sm.orders)     // 최종 결과 반환
        }
    }
}
```

핵심은 두 가지다:
1. **Continuation** 파라미터가 추가된다 — "어디까지 실행했고, 다음에 뭘 해야 하는가"를 담는 콜백 객체
2. **상태 머신(state machine)** 으로 변환된다 — `label`로 현재 상태를 추적하고, suspend point마다 상태를 전환한다

### Continuation — 콜백의 정체

`Continuation`은 Kotlin 표준 라이브러리에 정의된 인터페이스다:

```kotlin
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}
```

단 두 개의 멤버:
- `context` — 이 코루틴의 메타데이터를 담는 `CoroutineContext`. Dispatcher(실행 스레드), Job(생명주기), CoroutineName, ThreadContextElement 등 코루틴 로컬 정보를 포함한다
- `resumeWith()` — 결과를 받아 중단된 지점부터 실행을 재개한다

suspend 함수가 I/O를 기다리는 동안 일어나는 일을 시간 순서로 보면:

```
시간 ──────────────────────────────────────────────────────▶

Thread-1:
  ┌─ fetchUser() 시작 ─┐
  │  label=0            │
  │  fetchProfile() 호출 │
  │  → SUSPENDED 반환   │
  └─ 스레드 반납 ────────┘  ← Thread-1은 다른 코루틴 실행 가능
                    ...
                    ... (네트워크 I/O 완료)
                    ...
Thread-3:                   ← 같은 스레드일 수도, 다른 스레드일 수도 있음
  ┌─ continuation.resumeWith(profile) ─┐
  │  label=1                            │
  │  fetchOrders() 호출                  │
  │  → SUSPENDED 반환                   │
  └─ 스레드 반납 ───────────────────────┘

Thread-2:
  ┌─ continuation.resumeWith(orders) ─┐
  │  label=2                           │
  │  User(profile, orders) 반환        │
  └─ 완료 ─────────────────────────────┘
```

non-blocking suspend API에서 실제 suspension이 발생하면, 스레드는 suspend point에서 반납되고 I/O가 완료된 후 **아무 스레드에서나** `resumeWith()`가 호출되어 이어서 실행된다. 이것이 "스레드를 block하는 대신 suspend할 수 있다"의 정체다. 반대로 suspend 함수 내부에서 blocking I/O(JDBC 등)를 호출하면, suspension이 발생하지 않으므로 스레드는 그대로 점유된다.

### 상태 머신 변환의 의미

컴파일러가 상태 머신으로 변환하는 이유는 **스택 프레임을 힙으로 옮기기 위해서**다.

```
스레드 모델:
┌─────── Thread Stack (1MB) ───────┐
│  fetchUser() 스택 프레임          │  ← 스레드가 blocking되면
│    ├─ userId                     │     이 전체가 메모리에 묶임
│    ├─ profile (대기 중)          │
│    └─ orders  (아직 없음)        │
└──────────────────────────────────┘

코루틴 모델:
┌─ Continuation 객체 (Heap) ────────────┐
│  label = 1                              │
│  userId = 42                            │
│  profile = Profile(...)                 │  ← 힙에 저장. 스레드 불필요.
│  context = Dispatchers.IO               │
│  + Job, 캡처된 람다, 지역 변수 등        │
└─────────────────────────────────────────┘
```

- **스레드 스택**: blocking 중인 스레드가 살아 있어야 유지됨. 스택 전체가 메모리에 묶임.
- **Continuation 객체**: 힙에 저장. 재개에 필요한 상태(label, 지역 변수, context)만 보관. 스레드 없이도 존재.

코루틴도 Continuation 외에 Job, Context, 캡처된 람다 등 부가 메모리를 사용하지만, blocking 중인 스레드 스택 전체를 붙잡는 것에 비하면 훨씬 가볍다. 이것이 "코루틴 10만 개를 띄워도 메모리가 감당 가능한" 이유다.

---

## Dispatcher — 코루틴은 어디서 실행되는가

suspend 함수가 재개(resume)될 때, **어느 스레드에서** 실행될지를 결정하는 것이 `CoroutineDispatcher`다.

### 기본 제공 Dispatcher

| Dispatcher | 스레드 풀 크기 | 용도 |
|---|---|---|
| `Dispatchers.Default` | 사용 가능한 프로세서 수 (최소 2) | CPU 집약 작업 (정렬, 직렬화, 계산) |
| `Dispatchers.IO` | 최대 64개 (또는 코어 수 중 큰 값) | I/O 작업 (DB, HTTP, 파일) |
| `Dispatchers.Unconfined` | 호출한 스레드에서 시작, resume은 아무 데서나 | 테스트 용도. 프로덕션 비권장 |
| `Dispatchers.Main` | UI 메인 스레드 1개 | Android UI 업데이트 |

### Default vs IO — 같은 스레드 풀인가?

많이 오해하는 부분이다. `Dispatchers.Default`와 `Dispatchers.IO`는 내부적으로 **같은 스레드 풀을 공유**한다. 차이는 **동시 실행 허용 수**다.

```
┌─────────── 공유 스레드 풀 ───────────┐
│                                       │
│  ┌─ Default (제한: CPU 코어 수) ──┐   │
│  │  최대 4개 코루틴 동시 실행      │   │
│  └─────────────────────────────────┘  │
│                                       │
│  ┌─ IO (제한: 64개) ──────────────┐   │
│  │  최대 64개 코루틴 동시 실행     │   │
│  └─────────────────────────────────┘  │
│                                       │
│  실제 스레드는 필요에 따라 생성/공유   │
└───────────────────────────────────────┘
```

- `Default`: CPU 바운드 작업이 코어 수를 초과하면 context switching 오버헤드만 증가하므로 코어 수로 제한
- `IO`: I/O 대기 시간이 길어 스레드가 blocking되어도 다른 스레드에서 작업을 계속할 수 있으므로 넉넉하게 설정

### withContext — Dispatcher 전환

`withContext`는 코루틴의 `CoroutineContext`를 전환한다. Dispatcher가 달라지면 실행 스레드도 바뀔 수 있지만, Default와 IO처럼 스레드 풀을 공유하는 경우 실제로는 같은 스레드에서 실행될 수도 있다. 블록이 끝나면 원래 Context로 복귀한다.

```kotlin
suspend fun processProduct(productNo: Long): Product {
    // 현재: Dispatchers.Default에서 실행 중

    val product = withContext(Dispatchers.IO) {
        // Dispatchers.IO 스레드로 전환
        productRepository.findById(productNo)
    }
    // 다시 Dispatchers.Default로 복귀

    val enriched = withContext(Dispatchers.Default) {
        // CPU 집약 작업: Default에서 실행
        enrichProduct(product)
    }

    return enriched
}
```

스레드 흐름:

```
DefaultDispatcher-worker-1:
  ┌─ processProduct() 시작 ─┐
  │  withContext(IO) 호출     │
  └─ suspend ─────────────────┘

IO-worker-3:
  ┌─ productRepository.findById() ─┐
  │  DB 결과 반환                    │
  └─ resume ─────────────────────────┘

DefaultDispatcher-worker-2:        ← worker-1이 아닐 수 있음
  ┌─ enrichProduct() 실행 ─┐
  │  결과 반환               │
  └─ 완료 ──────────────────┘
```

주의할 점: `withContext` 전후로 **같은 스레드가 아닐 수 있다**. 이것은 `ThreadLocal`에 의존하는 코드에서 문제가 된다 (3편에서 다룸).

### limitedParallelism — 세밀한 동시성 제어

Dispatchers.IO의 64개 제한이 특정 작업에 부적절할 때, `limitedParallelism`으로 별도의 뷰를 만들 수 있다.

```kotlin
// DB 작업용: 커넥션 풀(20개)에 맞춰 동시 스레드 수 제한
val dbDispatcher = Dispatchers.IO.limitedParallelism(20)

// 무거운 파일 처리용: 디스크 경합 방지를 위해 동시 처리 수 제한
val fileDispatcher = Dispatchers.IO.limitedParallelism(4)

suspend fun queryDB() = withContext(dbDispatcher) {
    // 최대 20개 스레드에서 동시 실행
    productRepository.findAll()
}
```

`limitedParallelism`은 새로운 스레드 풀을 만드는 것이 아니라, 기존 풀에서 **동시에 사용하는 스레드 수를 제한하는 뷰(view)** 를 생성한다. 주의할 점이 두 가지 있다:

1. **스레드 수 제한이지 코루틴 수 제한이 아니다** — `limitedParallelism(10)`은 동시에 스레드를 점유하는 코루틴 수를 제한하는 것이지, suspend 중인 코루틴까지 포함한 전체 동시 코루틴 수를 제한하지 않는다. API rate limit처럼 동시 요청 수 자체를 제한하려면 `Semaphore`가 더 적합하다.
2. **IO의 기본 64 제한에 묶이지 않는다** — `Dispatchers.IO.limitedParallelism(n)`으로 만든 뷰는 IO의 기본 64개 제한과 독립적으로 **탄력적으로 확장**될 수 있다. `limitedParallelism(100)`은 IO 기본 제한과 별개로 최대 100개 스레드를 사용할 수 있다.

---

## 코루틴 vs 스레드 — 실제 차이를 코드로

### 10만 개 동시 실행

```kotlin
// 코루틴: 10만 개 동시 실행 — 정상 동작
fun coroutineTest() = runBlocking {
    val time = measureTimeMillis {
        val jobs = List(100_000) {
            launch {
                delay(1000L)  // suspend: 스레드 반납
            }
        }
        jobs.forEach { it.join() }
    }
    println("코루틴 10만 개 완료: ${time}ms")
    // 출력: 코루틴 10만 개 완료: ~1,050ms
}

// 스레드: 10만 개 동시 생성 — OOM 또는 극심한 지연
fun threadTest() {
    val time = measureTimeMillis {
        val threads = List(100_000) {
            Thread {
                Thread.sleep(1000L)  // blocking: 스레드 점유
            }.apply { start() }
        }
        threads.forEach { it.join() }
    }
    println("스레드 10만 개 완료: ${time}ms")
    // 결과: OutOfMemoryError 또는 수십 초 소요
}
```

### 메모리 비교

| | 코루틴 10만 개 | 스레드 10만 개 |
|---|---|---|
| 메모리 | ~수십 MB (Continuation 객체) | ~100 GB (스택 1MB × 10만) |
| 생성 비용 | 객체 할당 (수 μs) | OS 스레드 생성 (수백 μs ~ ms) |
| Context Switching | 코루틴 수준 전환 (매우 가벼움) | OS 수준 전환 (레지스터 저장/복원) |
| 동시 I/O 대기 | 10만 개 가능 | 스레드 풀 크기에 제한 |

### 핵심 차이: delay vs sleep

```kotlin
delay(1000L)          // suspend: 스레드 반납. 타이머만 등록.
Thread.sleep(1000L)   // blocking: 스레드 점유. 1초간 아무 일도 못 함.
```

`delay`는 현재 코루틴을 suspend 상태로 만들고 스레드를 반납한다. 1초 후 타이머가 만료되면 Dispatcher가 적절한 스레드에서 코루틴을 resume한다. 스레드 입장에서는 대기 시간이 **0**이다.

---

## 정리

| 개념 | 핵심 |
|---|---|
| suspend | 컴파일러에게 "이 함수는 중단 가능하다"고 알리는 표시 |
| CPS 변환 | suspend 함수에 `Continuation` 파라미터를 추가하고 상태 머신으로 변환 |
| Continuation | 중단 지점의 상태를 힙에 저장하는 객체. 재개 시 `resumeWith()` 호출 |
| 상태 머신 | `label`로 현재 상태를 추적. suspend point마다 상태 전환 |
| Dispatcher | 코루틴이 어느 스레드에서 실행될지 결정 (Default, IO, Unconfined) |
| withContext | Dispatcher를 전환. 블록 종료 후 원래 Dispatcher로 복귀 |

코루틴이 가벼운 이유는 단순하다. blocking 중인 스레드 스택 전체를 붙잡는 대신, 재개에 필요한 상태만 힙 객체로 보관하고 스레드를 반납한다. 컴파일러가 CPS 변환과 상태 머신을 만들어주니, 개발자는 동기 코드처럼 작성하면서 non-blocking suspension의 효율을 얻는다. 단, suspend 함수 내부에서 blocking I/O를 호출하면 이 이점은 사라진다 — suspend는 가능성이지 보장이 아니다.

다음 편에서는 이 코루틴들을 **누가 관리하는가** — `CoroutineScope`와 `CoroutineContext`의 관계, 그리고 **구조화된 동시성**이 왜 중요한지를 다룬다.
