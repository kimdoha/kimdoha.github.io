---
title: "캐시 톺아보기 (3) — 토폴로지와 실전 고려사항"
date: 2026-04-30 10:30:00 +0900
categories: [CS, Cache]
tags: [cache, local-cache, global-cache, distributed-cache, redis-cluster, consistent-hashing, reverse-proxy, caffeine, redis]
toc: true
---

---

## Application Cache 토폴로지

---

## Local Cache

다이어그램은 Server1, Server2, Server...n 각각이 내부에 독립적인 Cache와 App을 가진 구조를 보여준다.

```
┌─Server1─┐  ┌─Server2─┐  ┌─Server..n┐
│  Cache   │  │  Cache   │  │  Cache    │
│  App     │  │  App     │  │  App      │
└──────────┘  └──────────┘  └───────────┘
(서로 독립, 각자의 캐시)
```

**특징**:
- 각 서버가 자기 프로세스 메모리(Heap) 안에 캐시를 유지한다. (Part 1에서 다룬 CPU Cache(L1/L2/L3)는 CPU와 RAM **사이**에 위치하는 **하드웨어 캐시**지만, 여기서 말하는 Local Cache는 Caffeine/Guava 같은 **소프트웨어 캐시**로, Java 객체로서 JVM Heap 메모리 **안**에 저장된다. 계층이 다르다.)
- **장점**: 네트워크 I/O 없이 메모리에서 직접 접근하므로 **속도가 가장 빠르다**. Redis 왕복 ~0.5ms vs Local 접근 ~수 ns.
- **단점**:
  - 서버 간 캐시 데이터가 **불일치**할 수 있다 (Server1은 갱신됐지만 Server2는 옛날 데이터).
  - 서버 자원(메모리)을 소비한다 → 서버당 캐시 크기 제한.
  - 캐시 데이터 변경 시 다른 서버에 전파하려면 **clustering/replication** 필요 → scale-out 할수록 동기화 비용 증가.
- **대표 구현**: Caffeine (Java), Guava Cache, EhCache(local mode)
- **적합한 사례**: 변경이 거의 없고 모든 서버에 동일한 데이터가 필요한 경우 (설정값, 코드 테이블 등)

---

## Global Cache

다이어그램은 Server1, Server2, Server..n이 모두 **하나의 외부 캐시 서버**에 연결된 구조를 보여준다.

```
┌─Server1─┐
│  App     │──┐
└──────────┘  │
┌─Server2─┐  │    ┌──────────────┐
│  App     │──┼───▶│ Cache Server │
└──────────┘  │    │   (Redis)    │
┌─Server..n┐  │    └──────────────┘
│  App      │──┘
└───────────┘
(모든 서버가 하나의 캐시를 공유)
```

**특징**:
- 별도의 캐시 서버(Redis, Memcached 등)를 두고 모든 애플리케이션 서버가 접근한다.
- **장점**:
  - 서버 간 데이터 공유가 쉽다. 한 서버가 갱신하면 다른 서버도 동일한 최신 데이터를 읽는다.
  - 캐시 데이터 변경 시 추가 동기화 작업이 불필요하다.
  - Scale-out 할수록 효율이 좋다 (서버가 늘어나도 캐시는 1곳).
- **단점**:
  - 네트워크 I/O가 필요하므로 Local Cache보다 느리다.
  - 캐시 서버 장애 시 모든 서버에 영향.
- **데이터 분산**:
  - **Replication**: Master/Slave로 동일 데이터를 복제. 읽기 분산, 고가용성 확보.
  - **Sharding**: 같은 스키마의 데이터를 여러 노드에 분산 저장. 단일 노드 용량 한계 극복.

---

## Distributed Cache

다이어그램은 로드밸런서가 여러 서버로 트래픽을 분배하고, 각 서버가 분산된 캐시 클러스터(shard 0, 1, 2)에 접근하는 구조를 보여준다.

```
                          ┌─────────────────────────────┐
  LB ──▶ Server ──┐      │     Distributed Cache        │
  LB ──▶ Server ──┼─────▶│  [Shard0] [Shard1] [Shard2]  │
  LB ──▶ Server ──┘      └─────────────────────────────┘
```

**Global Cache와의 차이**:
- Global Cache는 단일(또는 Master-Slave 구조) 캐시 서버.
- Distributed Cache는 데이터를 여러 노드에 **샤딩**하여 분산 저장. 캐시 사이즈나 네트워크 용량이 단일 Global Cache의 한계를 넘을 때 사용한다.

**대표 구현**: **Redis Cluster**

Redis Cluster는 **Hash Slot** 방식으로 데이터를 분산한다. 전체 키 공간을 16384개의 슬롯으로 나누고, 각 노드가 슬롯 범위를 나누어 담당한다.

**슬롯 할당 공식**: `HASH_SLOT = CRC16(key) mod 16384`

```
예: 3개 노드 클러스터

  key: "user:1001"
  CRC16("user:1001") mod 16384 = 5765  →  Node B가 담당

┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Node A    │  │   Node B    │  │   Node C    │
│ slot 0-5460 │  │ slot 5461-  │  │ slot 10923- │
│             │  │      10922  │  │      16383  │
└─────────────┘  └──────┬──────┘  └─────────────┘
                        │
                  "user:1001" → slot 5765 → 여기에 저장
```

- **왜 16384개?**: 각 노드는 heartbeat 메시지로 자신이 담당하는 슬롯 비트맵을 전송한다. 16384bit = **2KB**로 네트워크 부담이 작다. 65536개면 8KB가 되어 부담이 커진다.
- **Hash Tag**: 키에 `{...}` 패턴이 있으면 중괄호 안의 문자열만 해시한다. 예: `{user:1001}:name`과 `{user:1001}:age`는 같은 슬롯에 배치 → 멀티키 연산 가능.
- **노드 추가/삭제**: 슬롯 범위를 재분배하고 해당 슬롯의 데이터만 이동한다. 전체 데이터 재배치가 아닌 **부분 마이그레이션**이므로 영향이 제한적이다.

---

## Consistent Hashing (일관된 해싱)

분산 캐시에서 노드가 추가되거나 삭제될 때 **키 재매핑을 최소화**하는 기법이다.

**일반 해싱의 문제**:
- `hash(key) % N`으로 노드를 결정. N이 변하면(노드 추가/삭제) 거의 모든 키가 다른 노드로 재매핑되어 캐시가 대량으로 무효화(cache stampede)된다.

**Consistent Hashing의 해결**:
- 노드와 키를 모두 동일한 **해시 링(hash ring)** 위에 배치한다.
- 키는 링 위에서 **시계 방향으로 가장 가까운 노드**에 매핑된다.
- 노드가 추가/삭제되면, 평균적으로 **K/n개의 키만 재매핑**된다 (K: 전체 키 수, n: 노드 수).
- 예: 100만 개의 키, 10개 노드 → 노드 1개 추가 시 약 10만 개(10%)만 이동. 일반 해싱이면 거의 100만 개가 이동.
- **실제 사용**: Twitter의 Twemproxy가 Redis/Memcached 앞단에서 Consistent Hashing으로 키를 분배한다.

---

## Reverse Proxy 캐시

다이어그램은 AWS의 DynamoDB Accelerator(DAX)를 예시로 보여주며, 이는 Read+Write Through 방식의 Reverse Proxy 캐시다.

```
Client ──▶ DAX Cluster(Cache) ──▶ DynamoDB
           (Read Through + Write Through)
```

**Reverse Proxy 캐시의 동작**:
- 클라이언트와 Origin 서버 사이에 위치하여, 클라이언트의 모든 요청을 가로챈다.
- **읽기(Read Through)**: 캐시에 있으면 반환, 없으면 Origin에서 가져와 캐시 후 반환.
- **쓰기(Write Through)**: 캐시에 먼저 기록하고, 캐시가 즉시 Origin에도 동기 반영한다.
- 클라이언트는 캐시의 존재를 알 필요가 없다 (**transparent**).

**대표 구현**:
- **Nginx microcaching**: Nginx가 reverse proxy로서 응답을 짧은 시간(1초 등) 캐싱. 트래픽 급증 시 Origin 부하를 극적으로 줄인다.
- **CDN** (CloudFront, Akamai 등): 전 세계에 분산된 엣지 서버가 콘텐츠를 캐싱.
- **DAX**: DynamoDB 전용 인메모리 캐시. API 호환이므로 애플리케이션 코드 변경 없이 적용 가능.

---

## Buffer vs Cache

| 구분 | Cache | Buffer |
|------|-------|--------|
| 목적 | **재사용** — 동일 데이터를 반복 접근할 때 속도 향상 | **완충** — 속도 차이가 큰 장치 사이에서 고속 장치의 대기를 줄임 |
| 데이터 보존 | 조회 후에도 삭제하지 않음 (TTL/Eviction 전까지 유지) | 한 번 소비(전달)하면 삭제 |
| 용량 | 상대적으로 작음 | 캐시보다 일반적으로 큼 |
| 대표 예시 | CPU L1~L3 캐시, Redis, CDN | 프린터 버퍼, I/O 버퍼, TCP 송수신 버퍼 |

**프린터 버퍼 예시**: PC가 인쇄 데이터를 프린터 버퍼에 넣으면 PC는 다른 작업을 계속할 수 있다. 프린터는 버퍼에서 데이터를 꺼내 인쇄한다. 버퍼가 없으면 프린터(저속 장치)가 인쇄를 마칠 때까지 PC(고속 장치)가 대기해야 한다.

---

## 캐시 적용 시 고려사항 정리

| 고려사항 | 설명 |
|---------|------|
| Capacity | 캐시에 얼마나 많은 데이터를 저장할 것인가? 파레토 법칙(20%)을 기준으로 산정 |
| Replacement Policy | LRU, LFU, FIFO 중 워크로드 특성에 맞는 정책 선택 |
| Hit Rate | 목표 적중률 설정 및 모니터링. 적중률이 낮으면 캐시 의미 없음 |
| Read-Write Strategies | 읽기/쓰기 비율, 일관성 요구 수준에 따라 전략 조합 |
| Coherence | 분산 환경에서 캐시 간 일관성 유지 방법 |
| Expiration | TTL 설정 — 너무 짧으면 Miss 증가, 너무 길면 stale 데이터 |
| Eviction | 용량 초과 시 어떤 항목을 제거할지 정책 설정 |

---

## 캐시 적용이 적합한/부적합한 케이스

**적합한 케이스**:
- 원본 데이터 접근 시간이 오래 걸리는 경우
- 반복적으로 동일한 결과를 반환하는 경우
- 업데이트가 자주 발생하지 않는 데이터
- 자주 조회되는 데이터 (Hot Data)
- 입력값과 출력값이 일정한 데이터

**부적합한 케이스**:
- **잦은 데이터 변경**: 캐시를 갱신하는 비용이 캐시의 이점을 상쇄
- **데이터 일관성이 실시간으로 보장되어야 하는 경우** (재고, 결제 잔액)
- **접근 패턴이 균등하여 Hot Data가 없는 경우** (hit ratio가 낮아짐)