---
title: "캐시 톺아보기 (4) — Eviction 정책, TTL 설계, 분산 락의 안전성"
date: 2026-05-20 10:00:00 +0900
categories: [CS, Cache]
tags: [cache, eviction, ttl, distributed-lock, caffeine, redis, w-tinylfu, redlock, countminsketch]
toc: true
---

---

> Part 1~3에서 캐시의 기본 개념, 읽기/쓰기 전략, 토폴로지를 다뤘다.
> Part 4에서는 **캐시가 가득 찼을 때 어떤 데이터를 버릴 것인가**(Eviction),
> **데이터를 얼마나 오래 유지할 것인가**(TTL),
> 그리고 **분산 락의 안전성 논쟁**(Redlock, Fencing Token)을 다룬다.

---

## 1. Caffeine의 Window TinyLFU

Part 3에서 Local Cache의 대표 구현으로 Caffeine을 소개했다. Caffeine이 높은 적중률을 달성하는 핵심은 **Window TinyLFU(W-TinyLFU)** eviction 정책이다.

---

### 왜 LRU만으로는 부족한가

LRU(Least Recently Used)는 가장 오래 전에 접근한 항목을 제거한다. 단순하지만 약점이 있다.

```
시나리오: 캐시 크기 3, 접근 순서 A B C A B C D A B C

LRU 캐시 상태 변화:
[A]  →  [A,B]  →  [A,B,C]  →  [B,C,A]  →  [C,A,B]  →  [A,B,C]
→ D 접근: A를 제거 → [B,C,D]  →  A 접근: Miss! B 제거 → [C,D,A]
→ B 접근: Miss! C 제거 → [D,A,B]  →  C 접근: Miss! D 제거 → [A,B,C]
```

- **scan 공격**: 한 번만 접근하는 대량의 데이터(예: 배치 처리, 풀스캔)가 캐시를 밀어내고, 자주 사용하는 hot 데이터가 제거된다.
- LFU(Least Frequently Used)는 빈도 기반이므로 scan에 강하지만, **과거 빈도에 갇혀** 최근 트렌드 변화에 적응하지 못한다.

---

### W-TinyLFU 구조

W-TinyLFU는 LRU의 **최신성**과 LFU의 **빈도 추적**을 결합한다.

```
New Entry ──▶ [ Window Cache (1%) ]
                     │
                     │ eviction candidate
                     ▼
              ┌──────────────┐
              │ TinyLFU Filter│ ◀── frequency sketch
              └──────┬───────┘
                     │
          ┌─ admit ──┤
          │          └─ reject ──▶ discard
          ▼
  [ Main Cache (99%) ]
  ┌──────────────────┬───────────────────┐
  │  Probation (20%) │  Protected (80%)  │
  └──────────────────┴───────────────────┘
```

**3개 영역**:
1. **Window Cache (~1%)**: 새로 들어온 항목이 최초로 위치하는 LRU 영역. burst 패턴을 흡수한다.
2. **Main Cache - Probation (~20%)**: Window에서 제거된 항목이 TinyLFU 필터를 통과하면 진입. 아직 검증 중인 영역.
3. **Main Cache - Protected (~80%)**: Probation에서 다시 접근된 항목이 승격. 가장 안전한 영역.

> 위 비율(1%/99%, 20%/80%)은 **기본 시작값**이다. Caffeine은 런타임에 **hill-climbing 최적화**로 Window와 Main의 비율을 워크로드에 맞게 적응적으로 조정한다.

---

### TinyLFU — 들어올 자격이 있는가?

Window Cache에서 제거된 항목(candidate)이 Main Cache에 입장하려면, Main Cache에서 제거 예정인 항목(victim)과 **빈도를 비교**한다.

```
candidate 빈도 > victim 빈도  →  candidate 입장, victim 제거
candidate 빈도 ≤ victim 빈도  →  candidate 버림, victim 유지
```

빈도 추적에는 **CountMinSketch**라는 확률적 자료구조를 사용한다.

---

### CountMinSketch

정확한 빈도를 저장하려면 모든 키에 대한 카운터가 필요하다 → 메모리 폭발. CountMinSketch는 **고정 크기 메모리**로 빈도를 **근사**한다.

**해시 함수 1개의 한계 — 충돌**

하나의 해시 함수로 고정 크기 배열에 빈도를 기록하면, 서로 다른 키가 같은 슬롯에 매핑(충돌)된다. 충돌이 발생하면 카운터에 다른 키의 접근 횟수까지 합산되므로, 실제보다 높게 추정된다.

```
해시 함수 1개, 배열 크기 8

  "A" (3회 접근) ──h1──▶ slot[2]
  "B" (7회 접근) ──h1──▶ slot[2]  ← 충돌!

  slot[2] = 10  →  "A" 빈도 조회 시 10 반환 (실제는 3)
```

**해법 — 독립된 해시 함수 d개 + 최솟값**

CountMinSketch는 d개의 독립된 해시 함수로 d개의 행을 운영한다. 각 해시 함수는 **서로 다른 충돌 패턴**을 만든다. "A"와 "B"가 h2에서 충돌하더라도, h1·h3·h4에서 모두 충돌할 확률은 극히 낮다.

```
"A" 3회, "B" 7회 접근 후 각 행의 상태:

  h1: [ ][ ][ ][3][ ][7][ ][ ]   A→col3, B→col5  (충돌 없음)
  h2: [ ][ ][ ][ ][ ][ ][ ][10]  A→col7, B→col7  (충돌! 3+7=10)
  h3: [ ][3][ ][ ][7][ ][ ][ ]   A→col1, B→col4  (충돌 없음)
  h4: [ ][ ][ ][ ][ ][3][7][ ]   A→col5, B→col6  (충돌 없음)

  "A" 빈도 = min(3, 10, 3, 3) = 3 ✓
                    ↑
              충돌 노이즈는 최솟값에서 걸러진다
```

카운터는 증가만 하고 감소하지 않으므로, 모든 행의 값은 **실제 빈도 이상**이다(과소 추정 불가). 따라서 최솟값을 취하면 충돌의 영향이 가장 적은 행의 값, 즉 **실제에 가장 가까운 상한**을 얻는다.

**정확도 조절**: 배열 너비(w)를 늘리면 충돌이 줄어 오차가 작아지고, 행 수(d)를 늘리면 보정 기회가 늘어 신뢰도가 올라간다. Caffeine은 4개 행(약 98% 신뢰도)을 사용하며, 캐시 admission은 정확한 빈도가 아닌 **상대적 크기 비교**만 필요하므로 이 정도면 충분하다.

- **공간**: O(ε⁻¹ × ln(δ⁻¹)). Caffeine은 엔트리당 약 8바이트를 사용하며, 4bit 카운터로 최대값 15까지만 추적한다.
- **노화(aging)**: 전체 카운터를 주기적으로 **우측 시프트(halving)** 시켜 과거 빈도의 영향을 줄인다. 이를 통해 트렌드 변화에 적응한다.

---

### 왜 Caffeine이 빠른가 (Eviction 외)

| 요소 | 설명 |
|------|------|
| 읽기 경합 최소화 | ConcurrentHashMap 기반, 정책 갱신을 버퍼링/일괄 처리하여 읽기 경로의 경합을 줄임 |
| Ring Buffer | 접근 이벤트를 ring buffer에 비동기 기록, 주기적으로 일괄 처리 |
| Amortized O(1) | 정책 적용(eviction, expiration)이 읽기/쓰기 연산에 분산되어 실행 |

> **참고 자료**
> - [Caffeine Wiki - Efficiency](https://github.com/ben-manes/caffeine/wiki/Efficiency) — 제작자(ben-manes)가 직접 작성한 공식 설명
> - [TinyLFU: A Highly Efficient Cache Admission Policy (arXiv)](https://arxiv.org/pdf/1512.00727) — 학술 근거
> - [Caffeine FrequencySketch.java 소스 코드](https://github.com/ben-manes/caffeine/blob/master/caffeine/src/main/java/com/github/benmanes/caffeine/cache/FrequencySketch.java) — 4bit CountMinSketch 구현

---

## 2. Redis maxmemory-policy (Eviction 정책)

Redis는 **인메모리** 저장소이므로, 메모리 한계(`maxmemory`)에 도달하면 어떤 키를 제거할지 결정해야 한다. 이 정책이 `maxmemory-policy`이다.

---

### 주요 8가지 정책

| 정책 | 대상 키 | 제거 기준 | 설명 |
|------|---------|----------|------|
| **noeviction** | - | 제거 안 함 | 메모리 초과 시 쓰기 명령에 에러 반환. 읽기는 정상 |
| **allkeys-lru** | 모든 키 | LRU | 가장 최근에 사용되지 않은 키 제거. **가장 보편적** |
| **allkeys-lfu** | 모든 키 | LFU | 가장 적게 사용된 키 제거. 빈도 기반 |
| **allkeys-random** | 모든 키 | 랜덤 | 무작위 제거 |
| **volatile-lru** | TTL 설정된 키만 | LRU | expire가 설정된 키 중 LRU 제거 |
| **volatile-lfu** | TTL 설정된 키만 | LFU | expire가 설정된 키 중 LFU 제거 |
| **volatile-random** | TTL 설정된 키만 | 랜덤 | expire가 설정된 키 중 무작위 제거 |
| **volatile-ttl** | TTL 설정된 키만 | TTL 짧은 순 | 만료가 가장 임박한 키 먼저 제거 |

> Redis 8.6에서 `allkeys-lrm`, `volatile-lrm` (Least Recently Modified — 읽기가 아닌 **쓰기 시점** 기준으로 제거)이 추가되어 현재 총 10가지 정책이 존재한다.

---

### 정책 선택 가이드

```
캐시 용도인가?
├── YES → 모든 키가 캐시 데이터?
│         ├── YES → allkeys-lru (기본 추천)
│         │         또는 접근 빈도 편차가 크면 allkeys-lfu
│         └── NO (일부는 영구 데이터) → volatile-lru
│                  (TTL 없는 키는 제거 대상에서 제외)
└── NO (영구 저장 용도) → noeviction
                          (메모리 초과 시 에러로 알림)
```

---

### Redis의 근사 LRU

Redis는 정확한 LRU를 구현하지 않는다. 정확한 LRU는 모든 키의 접근 시간을 정렬해야 하므로 메모리와 CPU 비용이 크다.

대신 **샘플링 기반 근사 LRU**를 사용한다:

```
1. maxmemory 도달
2. 무작위로 N개 키를 샘플링 (기본 N = 5, maxmemory-samples 설정)
3. 샘플 중 가장 오래된 키를 제거
4. 메모리가 확보될 때까지 반복
```

- `maxmemory-samples`를 늘리면 정확도가 올라가지만 CPU 비용도 증가한다.
- Redis 3.0부터 **eviction pool**을 도입하여 이전 라운드의 후보를 기억하고, 더 좋은 제거 대상을 선택한다.
- 실제 벤치마크에서 samples=10이면 정확한 LRU와 거의 동일한 적중률을 보인다.

---

### Redis LFU의 구현

Redis 4.0부터 LFU를 지원한다. 각 키에 8bit **로그 카운터**(0~255)를 유지한다.

```
카운터 증가 로직 (확률적):
  baseval = counter - LFU_INIT_VAL   // LFU_INIT_VAL = 5 (새 키의 초기 카운터)
  if baseval < 0: baseval = 0
  p = 1 / (baseval * lfu_log_factor + 1)
  if random() < p:
      counter++

카운터 감소 (시간 기반):
  minutes_since_last_access = (now - ldt) / lfu_decay_time
  counter -= minutes_since_last_access
```

- 새로 생성된 키는 **초기 카운터 5**(`LFU_INIT_VAL`)로 시작한다. 이 값이 있어야 새 키가 접근 이력을 쌓기 전에 즉시 evict되지 않는다.
- `lfu-log-factor` (기본 10): 높을수록 카운터 증가가 느려진다. factor=10이면 카운터가 255에 도달하려면 약 100만 회 접근이 필요하다.
- `lfu-decay-time` (기본 1): 1분마다 카운터를 1 감소시켜 최근 접근 패턴에 적응한다.

> **참고 자료**
> - [Redis 공식 문서 - Key eviction](https://redis.io/docs/latest/develop/reference/eviction/) — 정책 명세, LFU 카운터 동작
> - [Redis 소스 코드 - evict.c LFULogIncr](https://github.com/redis/redis/blob/unstable/src/evict.c) — LFU 카운터 증가 로직 원본

---

## 3. TTL 설계 가이드

TTL(Time To Live)은 키가 자동으로 만료되는 시간이다. Eviction이 "공간이 부족할 때" 작동하는 것과 달리, TTL은 "시간이 지나면" 해당 키를 **만료된 것으로 취급**한다. 실제 물리 삭제는 아래에서 설명하는 lazy/active expiration으로 처리된다.

---

### Redis의 만료 처리 메커니즘

Redis는 두 가지 방식을 조합하여 만료된 키를 제거한다.

**1. Lazy Expiration (수동적 만료)**
```
Client ──GET key──▶ Redis
                     │
                     ├── TTL 남아있음 → 값 반환
                     └── TTL 지남 → 키 삭제, nil 반환
```
- 접근할 때만 만료 여부를 확인한다.
- 접근되지 않는 만료 키는 메모리에 남아 있을 수 있다.

**2. Active Expiration (능동적 만료)**

단순화하면 다음과 같다 (실제 Redis 구현은 fast/slow cycle, expire effort 등 더 복잡하다):

```
매 100ms마다 (server.hz = 10 기준):
  1. TTL이 설정된 키 중 20개를 무작위 샘플링
  2. 만료된 키를 삭제
  3. 만료 키가 25% 이상이면 → 1번으로 돌아가 반복
  4. 25% 미만이면 → 다음 주기까지 대기
```
- 백그라운드에서 주기적으로 실행된다.
- 한 번에 너무 많이 삭제하면 블로킹이 발생할 수 있으므로, 임계값으로 반복 횟수를 제한한다.

> **참고 자료**
> - [Redis 공식 문서 - EXPIRE command](https://redis.io/docs/latest/commands/expire/) — Lazy/Active expiration 공식 명세
> - [Redis 소스 코드 - expire.c](https://github.com/redis/redis/blob/unstable/src/expire.c) — `ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP = 20`

---

### TTL 설계 원칙

| 원칙 | 설명 |
|------|------|
| **모든 캐시 키에 TTL 설정** | TTL 없는 캐시 키는 메모리 누수의 원인. Eviction에만 의존하면 예측 불가 |
| **데이터 변경 주기 기준** | 원본 데이터가 5분마다 변경 → TTL 5분 이하. stale 허용 범위에 맞추기 |
| **Jitter 추가** | 동일 TTL의 키가 동시에 만료되면 **Cache Stampede** 발생. TTL에 랜덤 ±10~20% 편차 부여 |
| **계층별 TTL 차등** | Local Cache TTL < Global Cache TTL. Local이 먼저 만료되어야 Global의 최신 데이터를 가져옴 |

---

### TTL 안티패턴

**1. TTL 없는 캐시 키**

```kotlin
// 안티패턴: TTL 없이 set
redis.set("product:1001", data)

// 올바른 패턴: 반드시 TTL 지정
redis.set("product:1001", data, Duration.ofMinutes(30))
```

TTL 없는 키는 명시적으로 `DEL`하지 않는 한 영원히 남는다. 데이터가 바뀌어도 캐시는 stale 상태로 유지된다. maxmemory에 도달할 때까지 메모리를 점유하며, `volatile-*` 정책에서는 eviction 대상에서도 제외된다.

**2. 동일 TTL 일괄 설정 (Cache Stampede)**

```kotlin
// 안티패턴: 1000개 키를 동시에 TTL 60초로 설정
keys.forEach { redis.set(it, data, Duration.ofSeconds(60)) }
// → 60초 후 1000개가 동시 만료 → DB에 1000개 쿼리 동시 발생

// 올바른 패턴: Jitter 추가
keys.forEach {
    val baseTtlSeconds = 60L
    val jitter = Random.nextLong(-12, 13)  // ±20% (60초 기준 ±12초)
    redis.set(it, data, Duration.ofSeconds(baseTtlSeconds + jitter))
}
```

**3. TTL 갱신 누락**

```kotlin
// 안티패턴: 데이터 수정 시 캐시만 삭제하고 관련 키는 무시
fun updateProduct(product: Product) {
    db.update(product)
    redis.delete("product:${product.id}")
    // 이후 읽기 시 Cache-Aside로 다시 적재됨
    // 하지만 관련 키(목록 캐시 등)의 TTL은 그대로 → stale
}

// 올바른 패턴: 관련 키도 함께 무효화
fun updateProduct(product: Product) {
    db.update(product)
    redis.delete("product:${product.id}")
    redis.delete("product-list:${product.categoryId}")  // 연관 캐시도 무효화
}
```

**4. 너무 긴 TTL**

TTL이 길수록 적중률은 올라가지만 stale 데이터 위험도 증가한다. 원본 데이터 변경 빈도와 비즈니스 허용 범위를 기준으로 설정해야 한다.

```
변경 빈도가 높은 데이터 (재고, 가격): TTL 10~30초
변경 빈도가 낮은 데이터 (카테고리, 설정): TTL 5~30분
거의 변경되지 않는 데이터 (코드 테이블): TTL 1~24시간
```

---

### Cache Stampede 방어 — Jitter 외 기법

Jitter는 가장 단순한 방어선이지만, 트래픽이 높은 환경에서는 추가 기법이 필요하다.

**Lock-based Recomputation**

캐시 미스가 발생하면 **1개 스레드만** DB를 조회하고, 나머지 스레드는 대기하거나 stale 값을 반환한다.

```kotlin
fun getProduct(id: Long): Product {
    val cached = redis.get("product:$id")
    if (cached != null) return cached

    // 락을 획득한 스레드만 DB 조회
    val lockValue = UUID.randomUUID().toString()
    val lockAcquired = redis.set("lock:product:$id", lockValue, NX, Duration.ofSeconds(5))
    if (lockAcquired) {
        try {
            val product = db.findById(id)
            redis.set("product:$id", product, Duration.ofSeconds(60))
            return product
        } finally {
            // Lua 스크립트로 내 락만 원자적 해제
            val script = "if redis.call('get',KEYS[1]) == ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end"
            redis.eval(script, listOf("lock:product:$id"), listOf(lockValue))
        }
    }

    // 락 획득 실패 → 짧은 대기 후 재시도
    Thread.sleep(50)
    return redis.get("product:$id") ?: db.findById(id)
}
```

**Refresh-Ahead (Stale-While-Revalidate)**

`refreshAfterWrite`와 `expireAfterWrite`를 조합하면, 만료 **전에** 백그라운드에서 값을 갱신하면서 기존 값을 즉시 반환한다.

```
Timeline:
0h            1h (refresh)               2h (expire)
├─────────────┼──────────────────────────┤
│  fresh hit  │  stale hit +             │  cache miss
│             │  background reload       │  (sync load)
```

```java
Caffeine.newBuilder()
    .maximumSize(5000)
    .refreshAfterWrite(Duration.ofHours(1))   // 1시간 후 → 백그라운드 갱신
    .expireAfterWrite(Duration.ofHours(2))    // 2시간 후 → 완전 만료
    .buildAsync(key -> loadFromDB(key));
```

- `refreshAfterWrite` 시점 이후 첫 접근 시, **기존 값을 즉시 반환**하면서 백그라운드 스레드가 새 값을 로딩한다.
- 요청 스레드가 블로킹되지 않으므로 latency spike가 없다.
- `expireAfterWrite`는 안전장치로, refresh가 실패하거나 접근이 없을 때 stale 데이터가 영원히 남는 것을 방지한다.
- **주의**: `refreshAfterWrite`는 `LoadingCache` 또는 `AsyncLoadingCache`에서만 사용 가능하다. `buildAsync(loader)`처럼 로딩 함수를 제공해야 한다.

이 패턴은 Local Cache(Caffeine)에서 주로 사용한다. Redis 같은 Global Cache에서는 직접 지원하지 않으므로, 애플리케이션 레벨에서 별도 구현이 필요하다.

**Probabilistic Early Expiration (XFetch)**

TTL 만료 **전에** 확률적으로 미리 갱신한다. 만료 시점에 가까울수록 갱신 확률이 높아진다.

핵심 아이디어는, 캐시를 읽을 때마다 "남은 TTL"과 "재계산 비용(delta)"을 고려한 확률 계산을 수행하여, 만료가 임박할수록 높은 확률로 캐시를 미리 갱신하는 것이다. 여러 서버 중 하나가 만료 전에 확률적으로 먼저 캐시를 갱신하므로, 만료 시점에 동시 요청이 DB로 몰리는 것을 방지한다.

> **참고 자료**
> - [Caffeine Wiki - Refresh](https://github.com/ben-manes/caffeine/wiki/Refresh) — refreshAfterWrite 공식 설명
> - [Optimal Probabilistic Cache Stampede Prevention (논문)](https://cseweb.ucsd.edu/~avattani/papers/cache_stampede.pdf) — XFetch 알고리즘 학술 근거

---

## 4. 분산 락 — 안전성 논쟁

분산 환경에서 여러 서버가 동일한 자원에 동시 접근할 때, **하나의 서버만 작업을 수행**하도록 보장하는 메커니즘이 분산 락이다.

기본 구현(`SET NX PX` + Lua 해제), Watchdog 패턴, Lettuce vs Redisson 비교 등 실무 패턴은 [OOM이 발생하는 두 가지 시나리오 — 동시성 제어 실패와 대량 데이터 처리](/posts/oom-lessons-from-failures/)에서 다뤘다. 여기서는 단일 Redis 락의 한계를 넘어서는 **Redlock 알고리즘과 그 안전성 논쟁**을 살펴본다.

---

### Redlock 알고리즘

단일 Redis 인스턴스의 분산 락은 그 Redis가 죽으면 락이 소실된다. **Redlock**은 N개(보통 5개)의 **독립된**(복제 관계가 아닌) Redis 인스턴스를 사용하여, 일부가 죽어도 락이 유지되도록 한다.

```
Redis A    Redis B    Redis C    Redis D    Redis E
  (독립)     (독립)     (독립)     (독립)     (독립)

Client가 TTL 10초로 락을 잡는 과정:

  T1 = 0.0초 (시작 시각 기록)
  │
  ├─ A에 SET NX PX 10000 → OK     (0.1초)
  ├─ B에 SET NX PX 10000 → OK     (0.2초)
  ├─ C에 SET NX PX 10000 → OK     (0.3초)
  ├─ D에 SET NX PX 10000 → 타임아웃 (D 장애)
  ├─ E에 SET NX PX 10000 → OK     (0.5초)
  │
  T2 = 0.5초 (종료 시각)

판정:
  ✅ 과반수 성공? → 4/5 성공 (3개 이상이면 OK)
  ✅ 락 획득에 걸린 시간 < TTL? → 0.5초 < 10초
  → 락 획득 성공

유효 잔여 TTL = 10초 - 0.5초 = 9.5초
  → A에는 이미 0.5초 전에 락을 걸었으므로, A의 TTL은 9.5초밖에 안 남았다.
  → 클라이언트가 실제로 작업할 수 있는 시간은 9.5초다.
```

락 획득에 실패하면(과반수 미달 또는 소요 시간 ≥ TTL) **모든 인스턴스에 DEL**을 보내 잔여 락을 정리한다.

---

### Redlock의 한계 (Martin Kleppmann의 비판)

Martin Kleppmann은 2016년 블로그에서 Redlock의 근본적 문제를 지적했다:

**1. Clock Drift 문제**
```
Redis A, B, C, D, E (5개 노드)

1. Client 1이 A, B, C에서 락 획득 (과반 성공)
2. C 노드의 시스템 시계가 앞으로 점프 → C의 TTL 조기 만료
3. Client 2가 C, D, E에서 락 획득 (과반 성공)
4. Client 1과 2가 동시에 락을 가진 상태 → Safety 위반
```

**2. GC Pause / Process Pause 문제**
```
1. Client 1이 락 획득 (TTL 10초)
2. Client 1에서 GC Pause 15초 발생
3. 10초 경과: 락 만료
4. Client 2가 락 획득 후 작업 수행
5. Client 1이 GC에서 돌아옴 → 자신이 아직 락을 가지고 있다고 믿음
6. 두 클라이언트가 동시에 자원 접근
```

**Kleppmann의 대안 — Fencing Token**:
```
┌────────────┐    ┌───────────┐    ┌──────────┐
│  Client 1  │    │ Lock Svc  │    │ Storage  │
└─────┬──────┘    └─────┬─────┘    └────┬─────┘
      │  lock request   │               │
      │────────────────▶│               │
      │  token=33       │               │
      │◀────────────────│               │
      │                 │               │
      │  (GC pause)     │               │
      │                 │               │
      │    Client 2 lock request        │
      │                 │◀──────────    │
      │           token=34              │
      │                 │──────────▶    │
      │                 │               │
      │ write(token=33) │               │
      │────────────────────────────────▶│
      │ REJECTED (33 < 34)              │
      │◀────────────────────────────────│
```

- 락 서비스가 락을 발급할 때마다 **1씩 증가하는 fencing token**을 함께 반환한다.
- Storage가 이전보다 낮은 token의 요청을 거부한다.
- 이 방식은 Redlock 없이도 안전하며, ZooKeeper의 zxid가 이 역할을 한다.

---

### Redis 창시자의 반론 (antirez)

antirez는 Kleppmann의 비판에 대해 다음과 같이 반박했다:

**Clock drift 반박**: 경과 시간 측정에는 monotonic clock(절대 뒤로 가지 않는 OS 시계)을 사용하므로, NTP 동기화에 의한 wall clock 점프가 TTL 계산에 영향을 주지 않는다.

**GC pause 반박**: 락 획득 과정에서 GC pause가 발생하면 T2-T1이 TTL을 초과하여 클라이언트가 스스로 "이 락은 유효하지 않다"고 판단한다. 단, **락 획득 이후에** GC pause가 발생하는 경우는 감지할 수 없다 — 이 한계는 Redlock뿐 아니라 모든 TTL 기반 락에 해당한다.

**Fencing token 반박**: Fencing token은 Storage 측이 "이전보다 낮은 token의 요청을 거부"하는 로직을 지원해야 한다. MySQL, 외부 API 등 대부분의 저장소가 이 기능을 기본 제공하지 않으므로, 직접 구현해야 하는 부담이 있다.

**결론적으로**: 이 논쟁에 명확한 승자는 없다. Redis는 성능과 가용성 중심의 시스템이므로, "동시에 두 클라이언트가 락을 가지는 상황"을 100% 방지하지는 못한다. 엄격한 정합성이 필요하면, 합의 알고리즘 기반인 ZooKeeper(ZAB)나 etcd(Raft)를 검토하는 것이 안전하다.

---

### 실무 선택 기준

| 요구사항 | 권장 방식 |
|---------|----------|
| 성능 중심, 간헐적 중복 허용 | 단일 Redis `SET NX PX` |
| 높은 가용성, 중복 최소화 | Redlock (5개 독립 인스턴스) |
| 강한 일관성, fencing token, 선형화 가능한 조정이 필요한 경우 | ZooKeeper / etcd (합의 알고리즘 기반) |

> **참고 자료**
> - [Redis 공식 문서 - Distributed Locks with Redis](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/) — Redlock 공식 스펙, Lua 스크립트 해제 패턴
> - [Martin Kleppmann - How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) — Redlock 비판, Fencing Token 제안
> - [antirez - Is Redlock safe?](http://antirez.com/news/101) — Redis 창시자의 반론

---

## 시리즈 정리

| Part | 주제 | 핵심 질문 |
|------|------|----------|
| 1 | 캐시 기본 개념, 읽기 전략 | 캐시란 무엇이고, 어떻게 읽는가? |
| 2 | 쓰기 전략 | 캐시에 어떻게 쓰는가? |
| 3 | 토폴로지, 실전 고려사항 | 캐시를 어디에 둘 것인가? |
| **4** | **Eviction, TTL, 분산 락의 안전성** | **캐시가 가득 차면? 데이터는 얼마나 유지? Redlock은 안전한가?** |
