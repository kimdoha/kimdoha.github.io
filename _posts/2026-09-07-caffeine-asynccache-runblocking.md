---
title: "캐시 미스 하나가 서비스를 멈추기까지 — Caffeine 동기 캐시와 runBlocking, 그리고 AsyncCache 전환기"
date: 2026-09-07 10:00:00 +0900
categories: [Kotlin, Coroutine]
tags: [kotlin, coroutine, caffeine, asynccache, runblocking, completablefuture, thread-pool, webflux, timeout, forkjoinpool, troubleshooting]
toc: true
---

운영 환경에서 간헐적으로 `RequestTimeout`(15초) 알림이 울리기 시작했습니다. 트래픽이 몰린 것도, 슬로우 쿼리가 있는 것도 아니었습니다. 분산 트레이스를 열어보니 더 이상했습니다. **15초짜리 스팬 하나에 하위 스팬이 0개**였거든요. DB 쿼리도, 외부 호출도 시작조차 하지 못한 채 정확히 15초를 채우고 타임아웃으로 종료되어 있었습니다.

이 글은 그 원인이었던 "동기 Caffeine 캐시 + `runBlocking`" 조합을 Caffeine `AsyncCache`로 전환한 과정의 기록입니다. 
전환 코드 자체는 열 줄이 채 안 되지만, 타임아웃이 있어도 왜 블로킹된 코드는 멈춰 세울 수 없는지, 캐시 로더가 어느 스레드에서 도는지, 그리고 executor 위에서 블로킹하면 무슨 일이 생기는지를 알아보겠습니다.

## TL;DR

- 동기 Caffeine 캐시의 로더 안에서 `runBlocking`으로 suspend 함수를 기다리면, 캐시 미스가 날 때마다 **요청 처리 스레드가 로딩이 끝날 때까지 블로킹**됩니다. 같은 키에 동시 미스가 나면 per-key 락 때문에 대기 스레드까지 전부 블로킹됩니다.
- 코루틴 취소는 강제 종료가 아니라, 코루틴이 suspend 지점에서 스스로 확인해야 동작하는 신호입니다. 그래서 `withTimeout`은 **suspend 중인 코루틴만** 구제합니다. 스레드째 블로킹된 요청은 타임아웃이 응답만 끊을 뿐, 스레드는 로딩이 끝날 때까지 계속 일합니다.
- 해법은 `Caffeine.buildAsync()` + `future { }` + `await()`입니다. 같은 키의 동시 미스는 하나의 `CompletableFuture`를 공유하고, 대기자는 스레드를 반납한 채 suspend 상태로 기다립니다.
- 단, AsyncCache의 로더는 **Caffeine executor(기본 `ForkJoinPool.commonPool`)에서 실행**됩니다. 로더 안에서 블로킹하면 요청 풀보다 훨씬 작고 JVM 전역이 공유하는 풀이 고갈됩니다. 따라서 로더는 executor 스레드를 점유하지 않아야 합니다 — 시작 즉시 suspend하거나, 블로킹 작업은 전용 디스패처로 옮겨 실행해야 합니다.

## 1. 증상 — 하위 스팬 0개짜리 15초

트레이스가 말해주는 사실은 두 가지였습니다.

- 문제 요청의 스팬은 **15002ms, 에러, 하위 스팬 0개.** 계측된 어떤 작업(DB SELECT, 외부 API 호출)도 시작하지 못했습니다.
- 같은 시간대의 다른 느린 트레이스들에는 전부 원인 스팬이 찍혀 있었습니다. 슬로우 쿼리면 SELECT 스팬이 길고, 외부 지연이면 HTTP 스팬이 길었습니다.

시작조차 못 했다는 건 스레드나 큐에 묶였다고 볼 수 있고, 게다가 15002ms라는 숫자는 요청 타임아웃 15초에서 정확히 2ms 넘친 값입니다. 즉 이 요청은 **타임아웃이 정상 작동해서 종료됐습니다.**

요청이 끝까지 실행권을 얻지 못했다는 건, 요청 처리 스레드들이 전부 다른 곳에 붙잡혀 있었다는 뜻입니다. 어디에 붙잡혀 있었는지 코드를 추적한 결과, 원인은 아래의 로컬 캐시 구현이었습니다. (코드는 공개용으로 재구성한 형태입니다.)

```kotlin
object LocalCacheManager {
    private val caches = CacheType.entries.associateWith {
        CaffeineCache(it.cacheName, Caffeine.newBuilder()
            .maximumSize(it.maximumSize)
            .expireAfterWrite(it.expiredTime)
            .build(), false)
    }

    suspend fun <KEY : Any, VALUE : Any> getData(
        type: CacheType, key: KEY, doOnMissMatch: suspend (KEY) -> VALUE,
    ): VALUE {
        return caches.getValue(type).get(key) {
            runBlocking {                      // ← 문제의 지점
                doOnMissMatch(key)
            }
        }!!
    }
}
```

`runBlocking`이 여기 들어간 이유는 분명합니다. 캐시 로더 `doOnMissMatch`는 suspend 함수인데, 동기 Caffeine의 로더 콜백은 일반 함수라서 suspend 함수를 직접 호출할 수 없습니다. 일반 함수 안에서 suspend 함수를 실행하는 가장 간단한 방법이 `runBlocking`이었던 겁니다.

문제는 `runBlocking`의 동작 방식입니다. `runBlocking`은 블록 안의 suspend 함수가 끝날 때까지 **호출한 스레드를 블로킹한 채 기다립니다.** suspend 함수의 목적은 "기다리는 동안 스레드를 풀에 반납하는 것"인데, `runBlocking`은 정반대로 그 스레드를 잡아두므로 suspend를 쓰는 의미가 사라집니다.

## 2. 원인 분석 — 세 가지 문제

### 2.1 캐시 미스마다 요청 처리 스레드가 블로킹된다

이 서비스는 WebFlux + 코루틴 구조입니다. 요청은 Netty 이벤트루프가 받고, 실제 처리는 고정 크기의 요청 디스패처에서 돕니다. 이 구조의 전제는 "suspend 함수는 기다리는 동안 스레드를 풀에 반납한다"는 것입니다.

`runBlocking`은 이 전제를 지키지 않습니다. 캐시 미스가 나면 로더가 끝날 때까지 — DB 커넥션 획득, 쿼리 실행, 결과 매핑까지 — **요청 디스패처 스레드 하나가 블로킹된 채 점유됩니다.** DB가 평소처럼 빠르면 문제가 드러나지 않지만, DB가 잠깐 느려지는 순간 블로킹된 스레드 수가 늘어나기 시작합니다.

### 2.2 같은 키를 기다리는 스레드도 전부 블로킹된다 (per-key 락)

동기 Caffeine의 `get(key, loader)`는 `ConcurrentHashMap.compute` 계열 위에 구현되어 있습니다. 같은 키에 대한 동시 미스는 **한 스레드만 로딩하고, 나머지는 맵 내부 락에서 기다립니다.** 캐시 스탬피드(thundering herd)를 막아주는 유용한 성질이지만, 이 기다림은 suspend가 아니라 **모니터 락 블로킹**입니다.

조회가 많은 캐시 키가 만료된 직후, 그 키를 원하는 요청이 동시에 10개 들어오는 상황을 가정해 보겠습니다. 1개는 `runBlocking`으로, 9개는 락 대기로 — 스레드 10개가 전부 블로킹됩니다. 문제의 캐시는 TTL이 짧고 `maximumSize`가 유난히 작은 반면, 키는 요청 파라미터 조합을 해시한 값이라 카디널리티가 높았습니다. 만료와 evict가 상시로 일어나는, 캐시 미스가 잦을 수밖에 없는 조합이었습니다.

### 2.3 타임아웃이 있어도 블로킹된 스레드는 풀리지 않는다

"그래도 요청마다 15초 타임아웃이 있으니 최악은 막아주지 않을까?" — 이 가정이 왜 틀리는지가 이 장애의 핵심입니다.

요청 진입점은 대략 이런 구조였습니다.

```kotlin
withTimeout(15_000) {
    withContext(requestDispatcher) {
        handler(request)
    }
}
```

코루틴 취소는 실행 중인 코루틴을 밖에서 강제로 멈추는 기능이 아닙니다. 취소는 신호를 보내는 것까지만 하고, **코루틴이 중단점(suspension point) — suspend 함수를 드나드는 지점 — 에 도달해 그 신호를 확인했을 때** 비로소 멈춥니다. Kotlin 공식 문서는 이 방식을 협조적 취소(cooperative cancellation)라고 부릅니다. 따라서 `withTimeout`이 만료되어도 취소 신호가 전달될 뿐이고, 중단점을 지나지 않는 코드는 그 신호를 확인할 수 없습니다. 이 구조에서 요청은 두 가지 상태 중 하나에 있습니다.

- **디스패처 큐에서 대기 중** (아직 스레드를 못 받음): suspend 상태이므로 취소가 즉시 적용됩니다. → 트레이스에서 본 15002ms짜리 하위 스팬 0개 요청이 정확히 이 케이스입니다.
- **`runBlocking`이나 락에 스레드째 블로킹됨**: 취소 신호는 전달되지만, 스레드가 블로킹되어 있어 suspend 지점에 도달하지 못하므로 코루틴이 신호를 확인하지 못합니다. 타임아웃은 **클라이언트 응답만 끊고**, 스레드는 로딩이 끝날 때까지 블로킹된 채 남습니다.

정리하면, 타임아웃은 느린 응답을 끊어줄 뿐 블로킹된 스레드를 회수하지는 못합니다. 오히려 응답이 끊긴 뒤에도 스레드는 계속 로딩을 수행하고 있으므로, 문제의 실체가 모니터링에서 잘 드러나지도 않습니다.

## 3. 장애 전파 구조 — 캐시와 무관한 API까지 타임아웃되는 이유

이 문제가 서비스 전체로 번지는 이유는 스레드풀 구조에 있습니다. 요청 처리 흐름은 다음과 같습니다.

```
┌────────────────────────────────────────────────────┐
│ ① Netty 이벤트루프 (~코어 수)                        │
│    요청 수락 · 응답 write — 블로킹 없음, 항상 생존     │
└──────────────┬─────────────────────────────────────┘
               │ withTimeout(15s) → withContext(요청 디스패처)
               ▼
┌────────────────────────────────────────────────────┐
│ ② 요청 처리 디스패처 (고정 크기, 수십 스레드)          │
│    ★ 수십 개 API 엔드포인트가 이 풀 하나를 공유        │
│                                                    │
│  캐시 만료 + DB 지연이 겹친 순간:                     │
│   T1      : runBlocking ████████ (로더)              │
│   T2 … Tk : per-key 락  ████████ (논서스펜드)        │
│   나머지   : 다른 키 미스·일반 처리로 잠식             │
│                                                    │
│   풀 전체 소진 → 이후 도착한 모든 요청(캐시와 무관한   │
│   엔드포인트 포함)이 큐에서 suspend 대기              │
│   → 15초 뒤 일괄 타임아웃                            │
└──────────────┬─────────────────────────────────────┘
               │ 트랜잭션 wrapper: withContext(DB 디스패처)
               ▼
┌────────────────────────────────────────────────────┐
│ ③ DB 디스패처 (커넥션풀 크기에 맞춰 사이징된 고정 풀)  │
│    블로킹 JDBC를 감당하라고 격리해둔 전용 풀           │
└────────────────────────────────────────────────────┘
```

이 구조 자체는 합리적입니다. 블로킹 JDBC는 ③에 격리하고, ②는 suspend 지점 사이의 짧은 실행만 담당합니다. 그런데 `runBlocking` 캐시 로더는 이 격리를 우회해 **②에서 직접 블로킹합니다.** ②는 모든 엔드포인트가 공유하는 단일 풀이므로, ②가 고갈되면 캐시와 무관한 API까지 큐에서 15초를 기다리다 타임아웃됩니다.

한편 ①은 계속 동작하므로 헬스체크는 통과하고, 부하가 줄면 스스로 회복합니다. 그래서 이 문제는 재현이 어렵고 원인 스팬도 남지 않는 **간헐 타임아웃** 형태로 나타나, 진단이 어렵습니다.

## 4. 해결 — AsyncCache + future + await 전환

Caffeine은 이런 상황을 위한 비동기 캐시 `AsyncCache`를 제공합니다. `build()` 대신 `buildAsync()`로 생성하며, 값 대신 `CompletableFuture`를 저장하기 때문에 로딩을 기다리는 쪽이 스레드를 블로킹하지 않습니다.

```kotlin
object LocalCacheManager {
    private val caches = CacheType.entries.associateWith {
        Caffeine.newBuilder()
            .maximumSize(it.maximumSize)
            .expireAfterWrite(it.expiredTime)
            .buildAsync<Any, Any>()            // ← Cache<K,V> 대신 AsyncCache<K,V>
    }

    @Suppress("UNCHECKED_CAST")
    suspend fun <KEY : Any, VALUE : Any> getData(
        type: CacheType, key: KEY, doOnMissMatch: suspend (KEY) -> VALUE,
    ): VALUE {
        return caches.getValue(type).get(key) { cacheKey, executor ->
            CoroutineScope(executor.asCoroutineDispatcher()).future {
                doOnMissMatch(cacheKey as KEY)
            }
        }.await() as VALUE
    }
}
```

변경된 코드는 몇 줄 되지 않지만, `buildAsync()`·`future { }`·`CoroutineScope`·`await()` 각각이 왜 필요한지 이해할 수 있었습니다.

### 4.1 `AsyncCache.get(key, mappingFunction)` — 값이 아니라 로딩 중인 `CompletableFuture`를 캐싱한다

AsyncCache의 `get`은 미스가 나면 `(key, executor)`를 건네며 `CompletableFuture<V>`를 요구하고, 반환된 future를 **완료되기 전에 즉시 맵에 넣습니다.** 이 한 가지 설계가 동기 캐시의 per-key 락 문제를 통째로 치환합니다.

- **같은 키의 동시 미스**: 두 번째 요청부터는 이미 맵에 있는 **미완료 future를 히트**합니다. 락 대기가 아니라, 같은 약속을 공유하고 각자 suspend 상태로 기다립니다. 로더는 정확히 1회만 실행됩니다.
- **로더 실패**: future가 예외로 완료되면 Caffeine이 **엔트리를 자동 제거**합니다. 실패가 캐시에 눌어붙지 않고, 다음 요청이 자연스럽게 재시도합니다.

"값을 캐싱"하던 구조가 "값에 대한 약속을 캐싱"하는 구조로 바뀌는 것이고, 대기의 성격이 스레드 블로킹에서 suspend로 바뀌는 지점이 바로 여기입니다.

### 4.2 `future { }` — suspend 함수를 블로킹 없이 `CompletableFuture`로 변환한다

일반 함수 안에서 suspend 함수를 실행하는 방법은 두 가지입니다. `runBlocking`은 suspend 함수를 실행하고 **결과값이 나올 때까지 호출 스레드를 블로킹한 뒤** 결과값을 반환합니다. 반면 `future { }`는 suspend 함수를 별도 코루틴으로 시작시키고 **호출 스레드를 붙잡지 않은 채 `CompletableFuture`를 즉시 반환합니다.** 결과값은 코루틴이 끝나는 시점에 그 future에 채워집니다.

| | `runBlocking { ... }` | `future { ... }` |
|---|---|---|
| 반환 시점 | suspend 함수가 끝난 뒤 | 호출 즉시 |
| 반환값 | 결과값 | `CompletableFuture` (완료 시 결과값이 채워짐) |
| 호출 스레드 | 완료까지 블로킹 | 블로킹 없음 |

이 자리에 어느 쪽이 맞는지는 콜백의 요구사항이 결정합니다. AsyncCache의 로더 콜백은 반환 타입이 `CompletableFuture<V>`입니다. 즉 완성된 값이 아니라 future를 요구하므로, 스레드를 블로킹하며 값을 만들어 반환하는 `runBlocking`이 아니라, future를 즉시 반환하는 `future { }`가 이 자리에 맞는 도구입니다.

### 4.3 `executor.asCoroutineDispatcher()` — 로더를 호출자 스레드와 분리해 실행한다

Caffeine이 건네주는 executor 위에서 로더 코루틴을 시작시킵니다. 로더 실행이 호출자(요청 디스패처) 스레드에서 분리되는 부분입니다.

### 4.4 `CoroutineScope(...)`를 매번 새로 만드는 이유 — 요청이 취소돼도 로딩은 완료되어야 한다

코루틴에서 스코프를 즉석에서 만드는 코드는 보통 리뷰에서 지적받는 안티패턴입니다. 하지만 여기서는 반대로 **의도된 설계**입니다.

로더를 호출자 코루틴의 자식으로 묶으면(구조적 동시성), 요청 하나가 타임아웃으로 취소될 때 로딩도 함께 취소됩니다. 문제는 그 로딩이 **다른 대기자들과 공유 중인 future**라는 점입니다. 먼저 온 요청이 죽었다고 뒤에 온 요청들의 로딩까지 죽이는 건 캐시의 의미론에 맞지 않습니다. 로딩은 요청과 독립적으로 완료되어 캐시에 남는 것이 맞습니다.

물론 트레이드오프도 있습니다. 로딩을 요청과 분리하면, 요청이 전부 취소되어도 **로딩은 중간에 취소할 방법 없이 끝까지 실행됩니다.** 로더가 짧은 DB 조회라면 이 비용은 문제가 되지 않습니다. 오히려 완료된 결과가 캐시에 남아 다음 요청이 히트하므로 이득입니다. 반대로 로더가 수십 초 걸리는 작업이라면, 아무도 기다리지 않는 작업이 자원을 계속 사용하는 셈이므로 이 설계를 그대로 쓰면 안 됩니다.

### 4.5 수정 후의 동작 흐름

```
요청 A (미스)                        요청 B~J (같은 키, 직후 도착)
   │                                 │
   ├─ get(key) → 미스                ├─ get(key) → 미완료 future 히트
   ├─ future 생성·맵에 등록          ├─ await() ── suspend, 스레드 즉시 반납
   ├─ await() ── suspend             │
   │   (요청 스레드 반납)            │
   │                                 │
   │   [executor] 로더 코루틴 → suspend 조회 → DB 디스패처
   │                                 │
   ├◀──── future 완료, 전원 재개 ───▶┤
   ▼                                 ▼
 응답                              응답
```

동기 캐시 시절 "1명 로딩 + 9명 락 블로킹 = 스레드 10개 소모"였던 그림이, "1개 로딩 + 10명 suspend = 스레드 0개 점유"로 바뀝니다.

## 5. 검증 — 구 구현이면 실패하는 동시성 테스트 작성

이번 전환은 반환값은 그대로인 채 스레드 사용 방식만 바뀌는 변경이라, 코드 리뷰만으로는 검증할 수 없습니다. 그래서 보장해야 할 동작 4가지를 동시성 테스트로 고정했습니다.

1. **캐시 히트 시 로더 미재실행** — 기본 계약
2. **같은 키 동시 미스 20건 → 로더 정확히 1회, 전원 같은 값** — future 공유 검증
3. **단일 스레드 디스패처에서 로딩 중에도 다른 코루틴이 실행된다** — 비블로킹 검증
4. **로더 예외는 원본 그대로 전파되고, 실패는 캐시되지 않아 다음 요청이 재시도한다** — 실패 의미론

핵심은 3번입니다. 이 테스트는 단순히 새 구현을 통과시키는 게 아니라, **구 구현이면 데드락으로 실패하는** 회귀 방지 장치입니다.

```kotlin
@Test
fun `단일 스레드 디스패처에서 캐시 로딩 중에도 다른 코루틴이 실행된다`() {
    newSingleThreadContext("single").use { dispatcher ->
        runBlocking(dispatcher) {
            val gate = CompletableDeferred<Unit>()

            val loading = async {
                LocalCacheManager.getData(CacheType.TEST, "key") {
                    gate.await()     // 로더가 끝나려면 아래 코루틴이 먼저 돌아야 한다
                    "loaded"
                }
            }
            // 구(runBlocking) 구현: 유일한 스레드가 로더에 잡혀 이 줄이 영원히 실행 불가 → 데드락
            // 신(AsyncCache) 구현: await()가 스레드를 반납하므로 이 줄이 실행됨
            gate.complete(Unit)

            assertEquals("loaded", loading.await())
        }
    }
}
```

이 테스트는 스레드가 1개뿐인 디스패처에서, 다른 코루틴(`gate.complete`)이 실행되어야만 로더가 끝날 수 있는 상황을 만듭니다. 구 구현은 유일한 스레드를 로더가 블로킹한 채 점유해 다른 코루틴이 실행될 수 없으므로 데드락에 빠지고, 신 구현은 `await()`가 스레드를 반납하므로 통과합니다.

4번 테스트에서는 예외 처리 방식의 차이도 확인했습니다. 기존에는 Spring `CaffeineCache`가 로더 예외를 `ValueRetrievalException`으로 래핑해 던졌지만, `AsyncCache`를 직접 쓰면 `await()`가 원본 예외를 그대로 던집니다. 그래서 `ValueRetrievalException` 타입에 의존하는 호출부가 있는지 확인한 뒤 진행했습니다.

## 6. 코드 리뷰에서 발견한 문제 — executor 스레드 위에서의 블로킹

전환을 마친 뒤, 코드 리뷰에서 중요한 문제를 발견했습니다.

캐시 사용처는 총 7곳이었고, 그중 6곳의 로더는 아래처럼 트랜잭션 wrapper로 DB 조회를 감싸고 있었습니다.

```kotlin
LocalCacheManager.getData(CacheType.X, key) {
    transactionHandler.runReadOnlyTransaction {   // suspend — 내부에서 DB 디스패처로 hop
        repository.query(...)
    }
}
```

그런데 딱 한 곳이 트랜잭션 wrapper 없이 블로킹 조회를 하고 있었습니다.

```kotlin
LocalCacheManager.getData(CacheType.Y, id) {
    repository.getView(id)     // non-suspend, 블로킹 JDBC
}
```

리뷰어의 지적은 이랬습니다. *"기존에는 runBlocking이라 어쨌든 요청 스레드에서 돌았지만, 전환 후에는 이 블로킹 쿼리가 Caffeine 기본 executor 위에서 돈다. 괜찮은가?"*

괜찮지 않았습니다. 코루틴은 **suspend 지점에서만** 스레드를 놓습니다. 로더 람다 안에 suspend 호출이 하나도 없으면, `future { }`로 시작한 코루틴이라도 첫 suspend를 만나기 전까지는 올라탄 스레드에서 그냥 동기 실행됩니다. 이 경우 그 스레드는 `ForkJoinPool.commonPool`의 워커이고, JDBC 소켓 read에서 그대로 잠들어 버립니다.

commonPool이 어떤 풀인지 생각하면 심각성이 보입니다. 기본 병렬도가 **(코어 수 − 1)**, 그리고 **JVM 전역 공유**입니다. `parallelStream`, executor를 지정하지 않은 `CompletableFuture` 조합 연산 등이 전부 이 풀을 씁니다. 즉 전환 전에는 블로킹이 수십 개짜리 전용 요청 풀에서 일어났지만, 전환 후에는 **스레드 수가 몇 개뿐이고 JVM 전체가 공유하는 commonPool에서 일어납니다.** 블로킹이 사라진 게 아니라, 더 작고 영향 범위는 더 넓은 풀로 옮겨간 것입니다.


```
수정 전:
 commonPool 워커 ──[코루틴 시작]──[JDBC 블로킹 ████████████]──[future 완료]
                                  ↑ suspend 지점 없음 = 워커가 통째로 인질

수정 후:
 commonPool 워커 ──[코루틴 시작]─[withContext ↴]              (μs 단위로 즉시 반납)
 DB 디스패처    ────────────────[JDBC ████████████]─[재개 → future 완료]
```

이 리뷰 덕분에 트랜잭션 wrapper가 하는 일이 두 가지라는 걸 명확히 인식하게 됐습니다. **트랜잭션 경계/리플리카 라우팅**, 그리고 **블로킹을 전용 풀로 격리**하는 것. 후자는 wrapper를 "DB 접근 규약"으로만 알고 쓰면 놓치기 쉬운 역할이었습니다.

정리하면 — **AsyncCache의 executor는 로더 코루틴을 시작시키는 용도이지, 작업을 수행하는 자리가 아닙니다.** 로더는 시작 즉시 suspend하거나 블로킹 작업을 전용 디스패처로 넘겨서, executor 스레드의 점유 시간을 코루틴을 시작시키는 데 드는 마이크로초 수준으로 유지해야 합니다.


## 7. 배운 것 — 같은 문제를 찾아내는 체크리스트

1. **트레이스에서 "하위 스팬 0개 + 타임아웃값과 거의 일치하는 지속시간"은 강력한 단서입니다.** 쿼리가 아니라 스레드풀과 큐를 의심하세요.
2. **타임아웃은 블로킹된 스레드를 회수하지 못합니다.** 코루틴 취소는 suspend 지점에서 확인되는 신호이므로, `withTimeout`이 취소할 수 있는 것은 suspend 중인 코루틴뿐입니다. "타임아웃이 있으니 최악은 막는다"는 가정은 블로킹 코드에는 적용되지 않습니다.
3. **suspend와 동기 콜백의 경계에서 `runBlocking`은 마지막 수단입니다.** 콜백이 future를 받을 수 있다면 해결책은 `future { }`입니다. — `runBlocking`은 "결과가 나올 때까지 정지", `future`는 "약속만 받고 즉시 진행".
4. **executor 위에서는 블로킹하지 않습니다.** Caffeine executor든 commonPool이든, 그 풀의 크기와 공유 범위를 모른 채 블로킹하면 병목이 그 풀로 옮겨갈 뿐입니다. 블로킹 IO는 그 용도로 사이징된 전용 디스패처에서 처리해야 합니다.

## References

- [Caffeine Wiki — Population: Asynchronous](https://github.com/ben-manes/caffeine/wiki/Population) — `AsyncCache`, 로더 future의 즉시 등록·실패 시 자동 제거 동작
- [Caffeine Wiki — Home](https://github.com/ben-manes/caffeine/wiki) — `Caffeine.executor` 설정, 기본값 `ForkJoinPool.commonPool`
- [Caffeine API — AsyncCache](https://www.javadoc.io/doc/com.github.ben-manes.caffeine/caffeine/latest/com.github.benmanes.caffeine/com/github/benmanes/caffeine/cache/AsyncCache.html) — `get(key, BiFunction<K, Executor, CompletableFuture<V>>)` 계약
- [kotlinx.coroutines — future builder](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.future/future.html) — suspend → CompletableFuture 브리지
- [kotlinx.coroutines — CompletionStage.await()](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.future/await.html) — 원본 예외 전파 동작
- [kotlinx.coroutines — runBlocking](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html) — "This function should not be used from a coroutine" 경고
- [Kotlin Docs — Cancellation and timeouts](https://kotlinlang.org/docs/cancellation-and-timeouts.html) — 취소의 협조성(cooperative cancellation), suspend 지점에서만 전달
- [Kotlin Docs — Coroutine context and dispatchers](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html) — `withContext`, `asCoroutineDispatcher`
- [JDK Javadoc — ForkJoinPool](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ForkJoinPool.html) — commonPool 기본 병렬도(코어−1), `ManagedBlocker`
- [Spring Framework — CaffeineCache](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/cache/caffeine/CaffeineCache.html) — 로더 예외의 `ValueRetrievalException` 래핑(전환 전 동작과의 차이)
- [HikariCP — About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing) — 커넥션 풀·DB 디스패처 사이징의 근거
