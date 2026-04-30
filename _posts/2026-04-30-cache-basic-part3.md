---
title: "캐시 톺아보기 (3) — 캐시 토폴로지"
date: 2026-04-30 10:30:00 +0900
categories: [CS, Cache]
tags: [cache, local-cache, global-cache, caffeine, redis, cache-topology]
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