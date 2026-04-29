---
title: "캐시 톺아보기 (2) — 쓰기 전략"
date: 2026-04-30 10:00:00 +0900
categories: [CS, Cache]
tags: [cache, write-around, write-back, write-through, cache-strategy]
toc: true
---

---

## Write Strategies (쓰기 전략)

---

## Write Around

다이어그램의 흐름:

**쓰기:**
```
App ──1:WRITE──▶ DB (캐시를 거치지 않고 DB에 직접 기록)
```

**읽기 (캐시에 있을 때):**
```
App ──2:READ(exists)──▶ Cache ──▶ 반환
```

**읽기 (캐시에 없을 때):**
```
App ──3-1:READ FROM DB(not exist)──▶ DB
App ◀─────────────────────────────── DB
App ──3-2:UPDATE──▶ Cache (읽은 데이터만 캐시에 저장)
```

**동작 원리**:
1. **쓰기**: DB에 직접 기록한다. 캐시에는 쓰지 않는다.
2. **읽기**: 캐시에 있으면 반환, 없으면 DB에서 읽어온 후 캐시에 저장한다.

**특징**:
- **읽은 데이터만 캐시에 저장**한다. 한 번 쓰고 거의 읽지 않는 데이터가 캐시 공간을 낭비하지 않는다.
- Cache-Aside, Read Through 전략과 결합하여 사용하는 것이 일반적이다.
- 쓰기 직후에는 캐시에 최신 데이터가 없으므로, 읽기 시 반드시 Miss가 발생한다 (stale 데이터 방지).
- **적합한 사례**: 한 번 쓰고 가끔 읽는 경우 (로그, 이력 등)

---

## Write Back (Write Behind)

다이어그램의 흐름:
```
App ──1:WRITE to cache constantly──▶ Cache (모든 쓰기를 캐시에 먼저)
                                     Cache ──2:write to db once in a while──▶ DB
                                     (일정 시간/조건 후 배치로 DB 반영)
```

**동작 원리**:
1. 모든 쓰기 요청을 **캐시에만** 기록한다. 애플리케이션에는 즉시 완료를 반환한다.
2. 캐시가 일정 시간 또는 일정 조건 충족 시 변경 사항을 **DB에 비동기로 일괄 반영(flush)** 한다.

**특징**:
- **쓰기가 매우 많은 경우 유리**하다. 매 쓰기마다 DB에 접근하지 않으므로 DB 쓰기 비용이 크게 감소한다.
- Read-Through와 결합하면, 최근 저장 및 액세스된 데이터를 항상 캐시에서 사용할 수 있다.
- **최대 위험**: 캐시 장애 시 아직 DB에 반영되지 않은 데이터가 **영구 소실**된다.
- **worst case**: Cache-Aside + Write-Back을 Redis로 구성 시, Redis 노드 장애 = 데이터 유실
- **실제 사용 예**: CPU의 Write-Back 캐시, OS의 Page Cache(dirty page flush), DB의 WAL 버퍼

---

## Write Through

다이어그램의 흐름:
```
App ──1:WRITE to cache──▶ Cache
                          Cache ──2:write to db immediately──▶ DB
                          (캐시 기록과 동시에 즉시 DB에도 반영)
```

**동작 원리**:
1. 쓰기 요청이 오면 **캐시에 기록**한다.
2. 캐시가 **즉시 DB에도 동일하게 기록**한다. 두 곳 모두 완료되어야 쓰기가 완료된다.

**특징**:
- **데이터 일관성(Coherence)을 보장**한다. 캐시와 DB가 항상 동일한 데이터를 가진다.
- Read Through + Write Through를 함께 적용하면, 읽기 캐시 이점 + 데이터 일관성을 동시에 얻는다.
- 캐시 크기 == DB 크기 (모든 쓰기가 양쪽에 반영되므로)
- **쓰기 지연 증가**: 캐시 + DB 양쪽에 쓰므로 Write Back 대비 느리다.
- **Write Back과의 차이**: Write Back은 "나중에 일괄 반영", Write Through는 "즉시 동기 반영"

---

## 전략 비교 요약

| 전략 | 읽기 | 쓰기 | 일관성 | 쓰기 성능 | 데이터 안전 |
|------|------|------|--------|----------|------------|
| Cache Aside | App→Cache→(Miss)→DB | App→DB (직접) | TTL/Invalidation | - | 높음 |
| Read Through | App→Cache→(Miss시 Cache가)→DB | - | TTL | - | 높음 |
| Write Around | - | App→DB (캐시 무시) | 높음 | 보통 | 높음 |
| Write Back | - | App→Cache, 나중에 DB | 낮음 | 매우 높음 | 낮음 (유실 위험) |
| Write Through | - | App→Cache→즉시 DB | 매우 높음 | 낮음 | 높음 |

실무에서는 단일 전략을 쓰지 않고, **Read 전략 + Write 전략을 조합**한다. 예: **Cache Aside + Write Around** (가장 보편적), **Read Through + Write Through** (일관성 중시)