---
title: "Spring WebFlux의 ArrayList 동시 접근으로 발생한 500 에러 — thread-safety 분석"
date: 2026-04-29 09:00:00 +0900
categories: [Spring, Concurrency]
tags: [spring-webflux, reactor, thread-safety, arraylist, kotlin-coroutines]
toc: true
---

---

## 들어가며

프로덕션에서 간헐적으로 500 에러가 올라왔다.

```
500 Server Error for HTTP GET "/api/products/123"
```

로그를 열어보면 `NoSuchElementException`인데, 비즈니스 코드가 한 줄도 없다. 전부 Reactor 내부 스택이다.

```
java.util.NoSuchElementException
  at java.util.ArrayList$Itr.next()
  at reactor.core.publisher.FluxIterable$IterableSubscription.slowPath()
  at reactor.core.publisher.FluxConcatMap$ConcatMapImmediate.onSubscribe()
  ...
  at reactor.core.publisher.FluxOnErrorResume$ResumeSubscriber.onError()
  ...
  at kotlinx.coroutines.reactor.MonoCoroutine.onCancelled()
  at kotlinx.coroutines.DispatchedTask.run()
```

ArrayList의 `next()`에서 `NoSuchElementException`이 발생했는데, 코드에서 ArrayList를 직접 쓴 적이 없다.

---

## 스택 트레이스를 3개 구간으로 나눠 읽기

이 스택은 한 번에 읽으면 헷갈리기 때문에, 아래에서 위로 3구간으로 나눠서 봐야 한다.

```
┌─────────────────────────────────────────────────┐
│ ③ 에러 응답 렌더링 중 FluxIterable에서 터짐       │  ← 로그에 찍힌 예외
│    ArrayList$Itr.next()                          │
│    FluxIterable.slowPath()                       │
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

①에서 코루틴 핸들러가 비즈니스 예외를 던진다. "존재하지 않는 상품" 같은 400 에러다.

②에서 이 예외가 Reactor 파이프라인을 타고 전파되면서 Spring의 에러 핸들러(`FluxOnErrorResume`)가 에러 응답 렌더링을 시작한다.

③에서 에러 응답을 클라이언트에 쓰는 과정에서 `FluxIterable`이 내부 `ArrayList`를 순회하다가 `NoSuchElementException`이 발생한다.

문제는 ③에서 발생한 2차 예외가 ①의 원본 예외를 **덮어쓴다**는 것이다. 에러 핸들러는 예외 타입으로 HTTP 상태를 결정하는데, `NoSuchElementException`은 커스텀 예외가 아니므로 `else → 500`으로 빠진다. 원래 400이어야 할 응답이 500이 된다.

---

## 이유: AbstractServerHttpResponse.commitActions

그러면 그 `ArrayList`가 무엇인지 확인해야 한다.

Spring WebFlux의 HTTP 응답 객체 내부를 보면 답이 나온다. `AbstractServerHttpResponse`에 `commitActions`라는 리스트가 있다.

```java
// AbstractServerHttpResponse.java — Spring Framework 5.x (Boot 2.x)
public abstract class AbstractServerHttpResponse implements ServerHttpResponse {

    private final List<Supplier<? extends Mono<Void>>> commitActions = new ArrayList<>(4);

    public void beforeCommit(Supplier<? extends Mono<Void>> action) {
        this.commitActions.add(action);
    }

    protected Mono<Void> doCommit(Supplier<? extends Mono<Void>> writeAction) {
        Flux.fromIterable(this.commitActions)
            .concatMap(Supplier::get)
            .then();
    }
}
```

`beforeCommit()`은 응답을 클라이언트에 flush하기 직전에 실행할 콜백을 등록하는 메서드다. 세션 저장, 쿠키 쓰기, CORS 헤더 설정 같은 것들이 여기 등록된다.

`doCommit()`은 이 콜백 리스트를 `Flux.fromIterable()`로 순회하면서 하나씩 실행한다.

그런데 `commitActions`가 **일반 `ArrayList`**다. `ArrayList`는 thread-safe하지 않다.

---

## thread-safe하지 않다는 것의 의미

`ArrayList`의 내부 구조는 단순하다.

```java
public class ArrayList<E> {
    Object[] elementData;   // 실제 데이터 배열
    int size;               // 현재 원소 개수
}
```

`add()`를 호출하면 아래와 같은 일이 일어난다.

```java
public boolean add(E e) {
    elementData[size] = e;   // 1) 배열에 값 쓰기
    size++;                  // 2) size 증가
}
```

단일 스레드에서는 문제가 없다. 1) 다음에 2)가 실행되기 때문이다. 하지만 **두 스레드가 동시에 접근하면** 상황이 달라진다.

### 명령어 실행 순서가 바뀔 수 있다

JVM과 CPU는 성능 최적화를 위해 명령어 실행 순서를 바꿀 수 있다. 단일 스레드 관점에서 결과가 동일하면 재배치해도 괜찮다고 판단하기 때문이다.

```
개발자가 쓴 코드               CPU가 실행하는 순서 (바뀔 수 있음)
────────────────              ─────────────────────────────
elementData[2] = C;           size = 3;            ← size가 먼저
size = 3;                     elementData[2] = C;  ← 값은 나중에
```

하나의 스레드 안에서는 어차피 둘 다 실행되므로 결과가 같다. 하지만 다른 스레드가 중간에 읽으면 `size=3`은 보이는데 `elementData[2]`에는 아직 값이 없는 상태를 볼 수 있다.

### CPU 캐시 가시성 문제

멀티코어 CPU는 코어마다 독립적인 캐시를 가지고 있어서, 한 코어에서 쓴 값이 다른 코어에 **즉시 보이지 않는다.**

```
      Core 1                              Core 2
  ┌──────────────┐                    ┌──────────────┐
  │   L1 Cache   │                    │   L1 Cache   │
  │  size = 3 ✓  │                    │  size = 3 ✓  │ ← size는 보임
  │  data[2] = C │                    │  data[2] = ? │ ← 값은 아직 안 보임
  └──────┬───────┘                    └──────┬───────┘
         │                                   │
         └──────────────┬────────────────────┘
                   ┌────┴────┐
                   │  메모리   │
                   └─────────┘
```

`synchronized`나 `volatile` 같은 동기화 장치가 있으면 캐시를 강제로 동기화한다. `ArrayList`에는 이런 동기화 장치가 없다. 이것이 thread-safe하지 않다는 의미다.

### 실제로 발생하는 시나리오

```
스레드 A (add)                      스레드 B (iterate)
────────────────                    ─────────────────

                                    iterator 생성 (size=2)
                                    next() → data[0] = A ✓
                                    next() → data[1] = B ✓

commitActions.add(action)
→ size가 3으로 변경

                                    hasNext() → cursor(2) < size(3) = true
                                    next() → data[2] 접근
                                           → 값이 아직 안 보임
                                           → NoSuchElementException
```

`hasNext()`가 `true`를 리턴했지만 `next()`에서 예외가 발생한다. `size`는 갱신됐지만 `elementData` 배열의 실제 데이터가 아직 이 코어에 보이지 않기 때문이다.

---

## WebFlux + 코루틴 환경에서 발생하는 이유

Spring WebFlux + Kotlin 코루틴 환경에서는 **두 종류의 스레드**가 동시에 하나의 응답 객체에 접근한다.

```
  Netty EventLoop 스레드             커스텀 스레드 풀 (NioDispatcher)
  ───────────────────               ─────────────────────────────

  요청 수신
  WebFilter 실행
  → beforeCommit() 등록
    (세션 저장 콜백 등)

  컨텍스트 전환 ─────────────────→   핸들러 실행
                                     비즈니스 로직
                                     → 예외 발생!

  에러 수신 ◄────────────────────   에러 전파
  에러 핸들러 진입
  → doCommit() 시작
  → Flux.fromIterable(commitActions)
    순회 시작

         ↑                           beforeCommit() 호출
         │                           → commitActions.add(action)
         │                           → 같은 ArrayList에 동시 접근!
         │
  → iterator.next()
  → NoSuchElementException
```

핵심은 `withContext`로 스레드 컨텍스트를 전환하기 때문에 `beforeCommit()`(add)과 `doCommit()`(iterate)이 서로 다른 스레드에서 실행된다는 것이다.

이 버그가 항상 발생하지 않는 이유도 여기에 있다. 두 스레드의 타이밍이 정확히 겹쳐야 하기 때문이다. 트래픽이 높으면 겹칠 확률이 올라가서, 간헐적으로 1~2건씩 올라오는 패턴이 전형적이다.

---

## Spring Boot 3.2에서의 수정

Spring Framework 6.1 (Boot 3.2)에서 `commitActions`가 `CopyOnWriteArrayList`로 변경되었다.

```java
// 변경 전 — Spring Framework 5.x
private final List<...> commitActions = new ArrayList<>(4);

// 변경 후 — Spring Framework 6.1
private final List<...> commitActions = new CopyOnWriteArrayList<>();
```

`CopyOnWriteArrayList`는 `add()` 할 때 내부 배열 전체를 복사한다.

```java
public boolean add(E e) {
    synchronized (lock) {
        Object[] newArray = Arrays.copyOf(array, len + 1);
        newArray[len] = e;
        setArray(newArray);   // 새 배열로 교체
    }
}
```

기존 iterator는 복사 전의 **원본 배열**을 계속 참조하고, 새 `add()`는 **새 배열**에서 일어난다. 두 스레드가 서로 다른 배열을 보기 때문에 간섭이 없다.

```
add() 이전:  iterator ──→ [A, B]        (원본 배열)
add() 이후:  iterator ──→ [A, B]        (여전히 원본)
             리스트   ──→ [A, B, C]     (새 배열)
```

Boot 3.2로 버전업하면 이 문제는 해결된다.
