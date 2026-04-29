---
title: "Spring WebFlux 500 에러의 정체 — ArrayList가 thread-safe하지 않아서 생긴 일"
date: 2026-04-29 09:00:00 +0900
categories: [Spring, Concurrency]
tags: [spring-webflux, reactor, thread-safety, arraylist, kotlin-coroutines]
toc: true
---

---

## 들어가며

프로덕션 환경에서 간헐적으로 500 에러가 올라왔다.

```
500 Server Error for HTTP GET "/api/products/123"
```

스택 트레이스를 열어보니 비즈니스 코드는 한 줄도 없고, Reactor 내부 코드뿐이었다.

```
java.util.NoSuchElementException
  at java.util.ArrayList$Itr.next()
  at reactor.core.publisher.FluxIterable$IterableSubscription.slowPath()
  at reactor.core.publisher.FluxIterable$IterableSubscription.request()
  at reactor.core.publisher.FluxConcatMap$ConcatMapImmediate.onSubscribe()
  ...
  at reactor.core.publisher.FluxConcatArray$ConcatArraySubscriber.onComplete()
  ...
  at reactor.core.publisher.FluxOnErrorResume$ResumeSubscriber.onError()
  ...
  at kotlinx.coroutines.reactor.MonoCoroutine.onCancelled()
  at kotlinx.coroutines.DispatchedTask.run()
  at java.util.concurrent.ThreadPoolExecutor.runWorker()
```

`ArrayList`의 `next()`에서 `NoSuchElementException`? 비즈니스 로직에서 ArrayList를 직접 쓴 적도 없는데?

이 글에서 원인을 끝까지 추적한다.

---

## 스택 트레이스 읽기

스택 트레이스를 3개 구간으로 나눠서 읽어야 한다.

```
┌─────────────────────────────────────────────────┐
│ ③ 에러 응답 렌더링 중 FluxIterable에서 터짐       │  ← 로그에 찍힌 예외
│    ArrayList$Itr.next()                          │
│    FluxIterable.slowPath()                       │
│    ...                                           │
│    FluxConcatArray$ConcatArraySubscriber          │
├─────────────────────────────────────────────────┤
│ ② Reactor 에러 전파 + 에러 핸들러 진입             │
│    FluxOnErrorResume.onError()                   │
│    MonoFlatMap.onError()                         │
│    MonoCreate$DefaultMonoSink.error()            │
├─────────────────────────────────────────────────┤
│ ① 코루틴 실행 → 예외 발생                         │  ← 원본 예외 (마스킹됨)
│    MonoCoroutine.onCancelled()                   │
│    DispatchedTask.run()                          │
│    ThreadPoolExecutor.runWorker()                │
└─────────────────────────────────────────────────┘
```

**①** 코루틴 핸들러에서 비즈니스 예외가 발생한다. 예를 들어 "존재하지 않는 상품" 같은 400 에러.

**②** 이 예외가 Reactor 파이프라인을 타고 전파되면서 `FluxOnErrorResume`(Spring의 에러 핸들러)이 에러 응답 렌더링을 시작한다.

**③** 에러 응답을 클라이언트에 쓰는 과정에서 `FluxIterable`이 내부 `ArrayList`를 순회하다가 `NoSuchElementException`이 터진다.

### 원본 예외가 사라지는 구조

문제는 ③에서 터진 2차 예외가 ①의 원본 예외를 **대체**한다는 것이다.

```
원본: CustomException (400 Bad Request)
  → 에러 핸들러 진입
  → 에러 응답 렌더링 중 NoSuchElementException 발생 (2차)
  → 최종 로그: NoSuchElementException (500 Internal Server Error)
```

에러 핸들러의 HTTP 상태 결정 로직:

```kotlin
fun determineHttpStatus(error: Throwable): HttpStatus = when (error) {
    is CustomException -> HttpStatus.BAD_REQUEST      // 원본이면 400
    // ...
    else -> HttpStatus.INTERNAL_SERVER_ERROR           // NoSuchElementException → 500
}
```

원래 400으로 처리될 예외가 500으로 변질된다.

---

## 원인: AbstractServerHttpResponse.commitActions

### Spring WebFlux의 응답 라이프사이클

```
요청 수신
  │
  ▼
WebFilter 체인          ← 필터가 beforeCommit() 등록 가능
  │
  ▼
핸들러 실행
  │
  ├── 성공 → 정상 응답 렌더링 ──┐
  └── 실패 → 에러 핸들러 ───────┤
                                ▼
                         doCommit() 실행
                           │
                           ▼
                     Flux.fromIterable(commitActions)
                           │
                           ▼
                     각 콜백 순차 실행 (세션 저장, 쿠키 쓰기 등)
                           │
                           ▼
                     응답 flush → 클라이언트
```

핵심은 `doCommit()` 단계다. Spring WebFlux의 `AbstractServerHttpResponse`는 응답을 클라이언트에 보내기 직전에 실행할 콜백 목록을 관리한다.

```java
// AbstractServerHttpResponse.java — Spring Framework 5.x (Boot 2.x)
public abstract class AbstractServerHttpResponse implements ServerHttpResponse {

    private final List<Supplier<? extends Mono<Void>>> commitActions = new ArrayList<>(4);

    public void beforeCommit(Supplier<? extends Mono<Void>> action) {
        this.commitActions.add(action);
    }

    protected Mono<Void> doCommit(Supplier<? extends Mono<Void>> writeAction) {
        // ...
        Flux.fromIterable(this.commitActions)
            .concatMap(Supplier::get)
            .then();
    }
}
```

`beforeCommit()`으로 등록되는 것들:

| 호출자 | 등록하는 작업 |
|--------|-------------|
| `DefaultServerWebExchange` | 세션 저장 |
| 쿠키 관련 컴포넌트 | 세션 쿠키 쓰기 |
| CORS 필터 | CORS 헤더 최종 설정 |
| 커스텀 WebFilter | 응답 헤더 추가 등 |

`doCommit()`이 이 목록을 `Flux.fromIterable()`로 순회하면서 각 콜백을 실행한다.

**문제: `commitActions`가 일반 `ArrayList`다.** 여러 스레드가 동시에 접근하면 깨진다.

---

## Thread-safe하지 않다는 것

### ArrayList 내부 구조

```java
public class ArrayList<E> {
    Object[] elementData;   // 실제 데이터 저장 배열
    int size;               // 현재 원소 개수
    int modCount;           // 구조적 수정 횟수 (add, remove 시 증가)
}
```

### add()는 원자적이지 않다

`add()` 연산은 내부적으로 여러 단계다.

```java
public boolean add(E e) {
    ensureCapacity(size + 1);   // 1) 배열 용량 확인/확장
    elementData[size] = e;      // 2) 배열에 값 쓰기
    size++;                     // 3) size 증가
    modCount++;                 // 4) 수정 카운터 증가
}
```

단일 스레드에서는 문제 없다. 하지만 다른 스레드가 **중간에 끼어들면** 상태가 꼬인다.

### CPU 명령어 재정렬

JVM과 CPU는 성능 최적화를 위해 **단일 스레드 관점에서 결과가 같다면** 실행 순서를 바꿀 수 있다.

```
개발자가 쓴 코드              CPU가 실제로 실행하는 순서 (바뀔 수 있음)
────────────────             ─────────────────────────────────
elementData[2] = C;          size = 3;            ← size가 먼저!
size = 3;                    elementData[2] = C;  ← 값은 나중에
```

단일 스레드에서는 어차피 둘 다 실행되니 결과가 같다. 하지만 다른 스레드가 중간에 읽으면 `size=3`은 보이는데 `elementData[2]`는 아직 안 써진 상태를 볼 수 있다.

### CPU 캐시 가시성

멀티코어 CPU는 코어마다 독립적인 캐시를 가진다. 한 코어에서 쓴 값이 다른 코어에 즉시 보이지 않는다.

```
      Core 1                              Core 2
  ┌──────────────┐                    ┌──────────────┐
  │   L1 Cache   │                    │   L1 Cache   │
  │  size = 3 ✓  │                    │  size = 3 ✓  │ ← size는 보임
  │  data[2] = C │                    │  data[2] = ? │ ← 값은 아직!
  └──────┬───────┘                    └──────┬───────┘
         │                                   │
         └──────────────┬────────────────────┘
                   ┌────┴────┐
                   │  메모리   │
                   │ size = 2 │  ← 아직 반영 안 됨
                   │ data = ? │
                   └─────────┘
```

`synchronized`나 `volatile` 같은 동기화 장치가 없으면 캐시 간 값 전파 시점이 보장되지 않는다.

### 타이밍별 시나리오

**정상 케이스 — 한 스레드만 접근:**

```
스레드 A (iterate)

iterator 생성 (size=2)
next() → data[0] = A  ✓
next() → data[1] = B  ✓
hasNext() → false
→ 정상 종료
```

**비정상 케이스 — 두 스레드가 동시 접근:**

```
스레드 A (add)                      스레드 B (iterate)
────────────────                    ─────────────────

                                    iterator 생성 (size=2 캐싱)
                                    next() → data[0] = A ✓
                                    next() → data[1] = B ✓

commitActions.add(action)
→ size = 2 → 3 (다른 코어에 전파)
→ data[2] = action (아직 전파 안 됨)
                                    hasNext() → cursor(2) < size(3) = true
                                    next() → data[2] 접근
                                           → 값이 없음!
                                           → NoSuchElementException 💥
```

`hasNext()`가 `true`를 리턴했는데 `next()`에서 터지는 이유다. `size`는 갱신됐지만 `elementData` 배열의 실제 데이터는 아직 보이지 않는 상태.

---

## WebFlux + 코루틴에서 왜 발생하는가

### 두 개의 스레드 풀

Spring WebFlux + Kotlin 코루틴 환경에서는 두 종류의 스레드가 관여한다.

```
┌─────────────────────────────────────────────────────┐
│                    요청 처리 흐름                      │
│                                                      │
│  Netty EventLoop ──┐                                 │
│  (I/O 스레드)       │  요청 수신                       │
│                    ▼                                 │
│              WebFilter 체인                           │
│              beforeCommit() 등록 ◄── 세션, 쿠키 등     │
│                    │                                 │
│                    ▼                                 │
│  ┌─── 컨텍스트 전환 (withContext) ───┐                 │
│  │                                  │                │
│  │  커스텀 스레드 풀 (NioDispatcher)  │                │
│  │  ┌────────────────────────────┐  │                │
│  │  │ 비즈니스 로직 실행            │  │                │
│  │  │ → DB 조회                   │  │                │
│  │  │ → 예외 발생!                 │  │                │
│  │  └────────────────────────────┘  │                │
│  │                                  │                │
│  └─── 에러 전파 (Reactor sink) ──────┘                │
│                    │                                 │
│  Netty EventLoop ◄─┘  에러 수신                       │
│                    │                                 │
│              에러 핸들러 진입                           │
│              → renderErrorResponse()                 │
│              → doCommit()                            │
│              → Flux.fromIterable(commitActions) ◄─── │
│                                                  │   │
│  커스텀 스레드 풀 ──────── beforeCommit() 호출 ────┘   │
│  (필터 콜백이 아직 실행 중)                             │
│                                                      │
│              → ArrayList 동시 접근                     │
│              → NoSuchElementException 💥             │
└─────────────────────────────────────────────────────┘
```

핵심은 **스레드 컨텍스트 전환** 때문에 `beforeCommit()`(add)과 `doCommit()`(iterate)이 서로 다른 스레드에서 실행된다는 것이다.

### 간헐적인 이유

이 버그는 항상 터지지 않는다. 두 스레드의 실행 타이밍이 정확히 겹쳐야 한다.

```
정상: add 완료 ─────────────── iterate 시작
                (시간 간격)

비정상: add 진행 중 ──┬── iterate 시작    ← 이 순간에만 발생
                      └── 겹침!
```

트래픽이 높을수록 스레드 스케줄링이 촘촘해지면서 겹칠 확률이 올라간다. 간헐적으로 1~2건씩 올라오는 패턴이 전형적이다.

---

## Spring Boot 3.2에서의 수정

### 변경 전 (Spring Framework 5.x / Boot 2.x)

```java
// 일반 ArrayList — 동기화 없음
private final List<Supplier<? extends Mono<Void>>> commitActions = new ArrayList<>(4);

public void beforeCommit(Supplier<? extends Mono<Void>> action) {
    this.commitActions.add(action);  // 그냥 add
}

protected Mono<Void> doCommit(...) {
    Flux.fromIterable(this.commitActions)  // 원본 리스트 직접 순회
        .concatMap(Supplier::get)
        .then();
}
```

### 변경 후 (Spring Framework 6.1 / Boot 3.2)

```java
// CopyOnWriteArrayList — 쓰기 시 배열을 복사
private final List<Supplier<? extends Mono<Void>>> commitActions = new CopyOnWriteArrayList<>();
```

`CopyOnWriteArrayList`는 `add()` 할 때 내부 배열 전체를 복사한다.

```java
public boolean add(E e) {
    synchronized (lock) {                              // 잠금 획득
        Object[] es = getArray();
        Object[] newElements = Arrays.copyOf(es, len + 1);  // 새 배열 복사
        newElements[len] = e;
        setArray(newElements);                         // 원자적으로 교체
    }                                                  // 잠금 해제
}
```

기존 iterator는 **복사 전의 배열**을 계속 참조한다. 새로운 `add()`는 **새 배열**에서 일어난다. 두 스레드가 서로 다른 배열을 보기 때문에 간섭이 없다.

```
add() 이전:  iterator ──→ [A, B]        (원본 배열)
add() 이후:  iterator ──→ [A, B]        (여전히 원본 — 안전)
             리스트   ──→ [A, B, C]     (새 배열)
```

이 구조에서는 `hasNext()`와 `next()` 사이에 다른 스레드가 `add()`를 해도 iterator가 보는 배열은 변하지 않는다.

---

## 정리

| 항목 | 내용 |
|------|------|
| **버그 위치** | `AbstractServerHttpResponse.commitActions` (plain `ArrayList`) |
| **증상** | `Flux.fromIterable(commitActions)` 순회 중 동시 `add()` → `NoSuchElementException` |
| **트리거** | 코루틴 핸들러 예외 → 에러 응답 렌더링(`doCommit`) 중 다른 스레드에서 `beforeCommit()` 호출 |
| **빈도** | 간헐적 (스레드 타이밍 의존, 트래픽 높을수록 확률 증가) |
| **영향** | 원본 400 에러가 500으로 변질 + 원본 예외 마스킹 |
| **해결** | Spring Boot 3.2+ 버전업 (`CopyOnWriteArrayList`로 변경됨) |

이 에러에서 배운 것 두 가지:

1. **Reactor 스택 트레이스에서 보이는 예외가 원본이 아닐 수 있다.** 에러 핸들러가 2차 예외를 던지면 원본 예외가 마스킹된다. 500이 찍혔다고 비즈니스 로직 버그라고 단정하면 안 된다.

2. **"thread-safe하지 않다"는 건 "터질 수도 있고 안 터질 수도 있다"는 뜻이다.** `ArrayList`의 동시 접근은 항상 터지는 게 아니라 스레드 타이밍이 겹칠 때만 터진다. 그래서 재현이 어렵고, 로그만 보고는 원인을 찾기 어렵다.
