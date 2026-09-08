---
title: "예외도 로그도 없이 멈춘 코루틴 — Job과 SupervisorJob의 한 줄 차이"
date: 2026-09-08 10:00:00 +0900
categories: [Kotlin, Coroutine]
tags: [kotlin, coroutine, supervisorjob, job, coroutinescope, structuredconcurrency, coroutineexceptionhandler, cancellation, troubleshooting]
toc: true
---

이벤트를 받아 색인을 돌리는 리스너가 있었습니다. 어느 날부터 이 리스너가 아무 일도 하지 않기 시작했습니다. API는 여전히 200을 반환하고, 예외 로그도 없고, 모니터링에도 아무것도 잡히지 않았습니다. 그런데 색인은 갱신되지 않았습니다. 서버를 재시작하면 다시 정상 동작했습니다.

원인은 스코프를 만드는 코드 한 줄이었습니다.

```kotlin
CoroutineScope(dispatcher)                        // 문제
CoroutineScope(SupervisorJob() + dispatcher)      // 정상
```

이 둘의 차이는 옵션 하나가 아니라 **자식의 실패가 부모를 죽이느냐 아니냐**입니다. 그리고 부모가 죽으면 이후의 `launch`는 예외도 로그도 없이 본문을 건너뜁니다.

## TL;DR

- `CoroutineScope(dispatcher)`는 Job이 없는 스코프가 아닙니다. 팩토리가 **일반 `Job()`을 자동으로 넣어줍니다.**
- 일반 Job은 자식이 실패하면 자기 자신도 취소됩니다. 한 번 취소된 Job은 되살아나지 않습니다.
- 취소된 부모 밑에서 `launch`를 부르면 새 코루틴은 **생성 즉시 취소**되고 본문이 실행되지 않습니다. 이건 실패가 아니라 취소라서 예외도 던져지지 않고 `CoroutineExceptionHandler`로도 보고되지 않습니다.
- `SupervisorJob`은 `childCancelled`가 `false`를 반환하는 것 하나만 다릅니다. 그래서 부모가 Active로 남습니다.
- 단, 직계 자식까지만입니다. 안쪽에 새 스코프가 생기면 그 안은 다시 일반 Job 규칙입니다.

## 1. 증상 — 실패가 아니라 침묵

장애의 특징은 이랬습니다.

- 색인 API는 계속 성공 응답을 반환합니다.
- 예외 로그가 없습니다. 스택트레이스도 없습니다.
- 색인만 갱신되지 않습니다.
- 재시작하면 회복되고, 시간이 지나면 다시 같은 상태가 됩니다.

"에러가 나서 멈췄다"면 로그를 따라가면 됩니다. 그런데 이건 **에러 없이 멈춘** 상태였습니다. 이 조합이 나오면 실행 흐름 자체가 시작되지 않았을 가능성을 봐야 합니다.

## 2. 소스로 확인한 메커니즘

kotlinx-coroutines 1.8.1 기준으로 네 단계를 따라가면 전부 설명됩니다.

### 2.1 팩토리가 Job을 자동으로 넣는다

가장 많이 오해하는 지점입니다. `CoroutineScope(dispatcher)`는 Job이 없는 스코프가 아닙니다.

```kotlin
public fun CoroutineScope(context: CoroutineContext): CoroutineScope =
    ContextScope(if (context[Job] != null) context else context + Job())
```

<small>`kotlinx/coroutines/CoroutineScope.kt`</small>

컨텍스트에 Job이 없으면 **일반 `Job()`을 만들어 붙입니다.** 즉 우리가 명시하지 않았을 뿐, 구조적 동시성의 부모가 이미 존재합니다.

### 2.2 자식이 실패하면 부모에게 신호를 보낸다

```kotlin
private fun cancelParent(cause: Throwable): Boolean {
    ...
    return parent.childCancelled(cause) || isCancellation
}
```

<small>`kotlinx/coroutines/JobSupport.kt`</small>

### 2.3 여기서 두 갈래로 갈린다

일반 Job의 기본 구현은 신호를 받아 **자기 자신을 취소**합니다.

```kotlin
public open fun childCancelled(cause: Throwable): Boolean {
    if (cause is CancellationException) return true
    return cancelImpl(cause) && handlesException
}
```

<small>`kotlinx/coroutines/JobSupport.kt`</small>

`SupervisorJob`은 이 메서드 하나만 다릅니다. 클래스 전체가 두 줄입니다.

```kotlin
private class SupervisorJobImpl(parent: Job?) : JobImpl(parent) {
    override fun childCancelled(cause: Throwable): Boolean = false
}
```

<small>`kotlinx/coroutines/Supervisor.kt`</small>

`false`를 돌려주므로 `cancelImpl`이 아예 호출되지 않고, 부모는 Active로 남습니다. 대신 예외를 부모가 인수하지 않으므로 자식이 직접 `handleCoroutineException`으로 넘깁니다.

### 2.4 취소된 부모 밑의 launch는 조용히 건너뛴다

여기가 "왜 로그가 없었는가"의 답입니다. 취소된 부모 밑에서 `launch`를 부르면 새 코루틴은 생성 즉시 취소되고 본문이 실행되지 않습니다. 그런데 이건 **취소이지 실패가 아닙니다.** 그래서 호출부에 예외가 던져지지도 않고, `CoroutineExceptionHandler`로 보고되지도 않습니다.

API가 계속 성공 응답을 반환하면서 아무 일도 하지 않은 게 정확히 이 상태입니다.

## 3. 관측되는 동작 차이

| 시점 | `Job()` (기본값) | `SupervisorJob()` |
|---|---|---|
| 첫 예외 직후 `scope.isActive` | `false` (영구) | `true` |
| 예외 전달 경로 | 부모가 인수 → 스코프 실패 | 자식이 `CoroutineExceptionHandler`로 |
| 같이 돌던 형제 코루틴 | 전부 취소됨 | 영향 없음 |
| 이후 `launch` 본문 실행 | 안 됨 | 정상 |
| 이후 `launch` 호출부 반환 | 예외 안 던짐, 이미 취소된 Job 반환 | 정상 Job 반환 |
| 회복 수단 | 애플리케이션 재시작 | 불필요 (자가 회복) |
| `scope.cancel()` 했을 때 | 둘 다 동일 — 자식 전부 취소 | 둘 다 동일 |

마지막 줄이 중요합니다. **`SupervisorJob`은 아래에서 위로 올라오는 실패만 막습니다.** 위에서 아래로 내리는 취소는 그대로 전파됩니다.

## 4. SupervisorJob이 막아주지 않는 것

**직계 자식까지만입니다.** `launch { coroutineScope { launch { … } } }`처럼 안쪽에 새 스코프가 생기면 그 안은 다시 일반 Job 규칙입니다. 안쪽 자식의 실패는 안쪽 부모를 취소시킵니다.

**예외를 없애주지 않습니다.** `try/catch`가 없으면 예외는 `CoroutineExceptionHandler`로 가고, 지정하지 않았다면 스레드 기본 핸들러를 거쳐 stderr로 나갑니다. 로거를 타지 않으므로 운영 로그에 안 남을 수 있습니다.

**순서는 무관합니다.** `SupervisorJob() + dispatcher`든 반대든 같습니다. `CoroutineContext` 덧셈은 키별 병합이라 Job 키와 Dispatcher 키가 각각 들어갑니다.

**생명주기는 별개 문제입니다.** 싱글톤 필드로 들고 있는 스코프는 애플리케이션과 함께 영원히 삽니다. 종료 시 정리(`@PreDestroy`에서 `cancel()`)가 없으면 진행 중인 작업이 중단 처리 없이 끊깁니다.

## 5. 덤으로 배운 것 — 두 수정이 서로를 가릴 때

이 수정과 함께 리스너 본문에 `try/catch`도 들어갔습니다.

```kotlin
scope.launch {
    try { indexingService.index(event.targets) }
    catch (e: Exception) { log.error("indexing failed", e) }
}
```

그런데 이 둘을 같이 넣으면 **`try/catch`가 예외를 먼저 삼키므로 2.2의 `cancelParent` 자체가 시작되지 않습니다.** `childCancelled`도 호출되지 않습니다. 즉 이 상태에서는 `SupervisorJob`이 있든 없든 스코프가 살아남습니다.

확인해 보니 실제로 그랬습니다. **`SupervisorJob()`만 지우고 `try/catch`를 남기면 새로 추가한 테스트가 전부 통과합니다.** 핵심 변경이 회귀망 밖에 있다는 뜻입니다.

실무적으로는 둘 다 두는 게 맞습니다. `try/catch`는 이 리스너의 예외를 로거로 보내고, `SupervisorJob`은 나중에 누군가 `try/catch` 밖에서 코드를 추가했을 때의 안전망입니다. 다만 **그 안전망이 테스트로 지켜지지 않는다는 사실은 기록해 둬야 합니다.** 안 그러면 다음 사람이 "이거 없어도 테스트 통과하네"라며 지웁니다.

## 6. 정리 — 같은 문제를 찾아내는 체크리스트

1. **"예외 없이 조용히 멈춤"은 취소를 의심하세요.** 실패는 로그를 남기지만 취소는 남기지 않습니다.
2. **`CoroutineScope(dispatcher)`에는 이미 일반 Job이 들어 있습니다.** 명시하지 않았다고 부모가 없는 게 아닙니다.
3. **오래 사는 스코프에는 `SupervisorJob`을 명시하세요.** 이벤트 리스너, 백그라운드 워커처럼 애플리케이션 수명과 함께 가는 스코프가 대상입니다. 요청 단위로 만들고 버리는 스코프는 해당하지 않습니다.
4. **`SupervisorJob`은 직계 자식까지입니다.** 중첩 스코프 안쪽은 다시 일반 규칙입니다.
5. **`try/catch`와 `SupervisorJob`은 역할이 다릅니다.** 전자는 이 코드의 예외를 처리하고, 후자는 처리 못 한 예외가 스코프를 죽이지 않게 합니다. 둘 다 두되, 서로를 가려 테스트가 무력해지지 않는지 확인하세요.

## References

- [kotlinx.coroutines — CoroutineScope factory](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/CoroutineScope.kt) — 컨텍스트에 Job이 없으면 `Job()`을 붙이는 부분
- [kotlinx.coroutines — JobSupport](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/JobSupport.kt) — `cancelParent`, `childCancelled` 기본 구현
- [kotlinx.coroutines — Supervisor](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/Supervisor.kt) — `SupervisorJobImpl`
- [Kotlin Docs — Coroutine exceptions handling](https://kotlinlang.org/docs/exception-handling.html) — 예외 전파와 `SupervisorJob`
- [Kotlin Docs — Cancellation and timeouts](https://kotlinlang.org/docs/cancellation-and-timeouts.html) — 취소가 협조적으로 동작하는 방식
- [Kotlin Docs — Coroutine context and dispatchers](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html) — `CoroutineContext` 키별 병합
