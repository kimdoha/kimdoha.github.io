---
title: "캐시 톺아보기 (1) — 개념과 읽기 전략"
date: 2026-04-29 10:00:00 +0900
categories: [CS, Cache]
tags: [cache, cpu-cache, memory-hierarchy, cache-aside, read-through, eviction-policy, pareto]
toc: true
---

---

## 들어가며

캐시는 어디에나 있다. CPU 내부, 브라우저, CDN, Redis까지. 하지만 "왜 캐시가 필요한가"에 대한 근본적인 이해 없이 사용하면 오히려 독이 될 수 있다. 이 글에서는 캐시가 존재하는 이유를 하드웨어 계층부터 시작해서, 캐시 교체 정책과 읽기 전략까지 정리한다.

---

## 컴퓨터 메모리 계층 구조 (Hierarchy of Computer Memory)

다이어그램은 피라미드 형태로 컴퓨터의 저장 장치를 **위(빠른 것)에서 아래(느린 것)** 순으로 배치한다.

| 계층 | 속도 | 용량 | 비용 |
|------|------|------|------|
| CPU (레지스터) | 가장 빠름 | 수 Byte~KB | 가장 비쌈 |
| CPU Cache (L1/L2/L3) | 매우 빠름 | KB~수십 MB | 매우 비쌈 |
| Physical Memory (RAM) | 빠름 | GB 단위 | 보통 |
| Solid State Memory (SSD) | 보통 | 수백 GB~TB | 저렴 |
| Virtual Memory / HDD | 느림 | TB 단위 | 저렴 |
| External Network | 가장 느림 | 사실상 무한 | 가변적 |

**핵심 원리**: 피라미드 위로 올라갈수록 **속도는 빨라지고, 용량은 줄어들며, 단위당 가격은 비싸진다**. 캐시의 존재 이유가 바로 이 trade-off에 있다. CPU가 필요한 데이터를 매번 느린 하위 저장소에서 가져오면 병목이 발생하므로, 자주 사용하는 데이터를 빠른 상위 계층에 임시 저장하는 것이 캐시의 본질이다. 캐시는 **참조 지역성(Locality of Reference)** 의 특성을 활용한다. 공간 지역성(spatial locality)은 어떤 주소에 접근하면 근처 주소에도 곧 접근할 가능성이 높다는 것이고, 시간 지역성(temporal locality)은 최근 접근한 데이터가 가까운 미래에 다시 접근될 가능성이 높다는 것이다. 이러한 성격으로 캐시는 **가장 최근에 접근했거나, 가까이 있는 데이터를 우선 저장**한다.

---

## CPU Cache & Memory (CPU 캐시와 메모리 구조)

다이어그램은 4개의 CPU 코어 내부 구조와 공유 캐시, 메인 메모리, 스토리지 간의 관계를 보여준다.

```
┌─────────────────── CPU ───────────────────┐
│  Core 1    Core 2    Core 3    Core 4     │
│ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐  │
│ │Register│ │Register│ │Register│ │Register│ │  ~1KB, 1-2 cycles
│ │L1 Cache│ │L1 Cache│ │L1 Cache│ │L1 Cache│ │  ~100KB, 2-3 cycles
│ │L2 Cache│ │L2 Cache│ │L2 Cache│ │L2 Cache│ │  ~500KB, 3-5 cycles
│ └───────┘ └───────┘ └───────┘ └───────┘  │
│         ┌─── L3 Cache (공유) ───┐         │  ~20MB, 30-50 cycles
└─────────────────────────────────────────────┘
          ┌─── Main Memory ──────┐              GB, 50-200 cycles
          ┌─── SSD / HDD ───────┐              TB, 50000 cycles
```

각 계층별 상세 설명:

- **Register (~1KB, 1-2 cycles)**: CPU가 현재 처리 중인 데이터를 담는 가장 작고 빠른 저장소. 변수 하나, 명령어 하나 수준의 크기다.
- **L1 Cache (~100KB, 2-3 cycles)**: 각 코어에 전용으로 내장된다. 보통 **데이터용(L1d)**과 **명령어용(L1i)**으로 분리된다. 코어가 가장 먼저 참조하는 캐시다.
- **L2 Cache (~500KB, 3-5 cycles)**: 역시 코어별로 존재하지만, L1보다 용량이 크고 약간 느리다. CPU 벤더에 따라 여러 코어가 L2를 공유하는 구조도 있다 (예: Apple M2 Pro).
- **L3 Cache (~20MB, 30-50 cycles)**: 모든 코어가 **공유**하는 캐시. 한 코어에서 처리한 데이터를 다른 코어에서도 접근할 수 있게 한다. 일부 프로세서에서는 L3가 프로세서 내부에 없고 메인보드에 위치하거나 아예 없는 경우도 있다.
- **Main Memory (GB, 50-200 cycles)**: DRAM. 프로그램 실행에 필요한 데이터와 코드가 적재되는 곳이다.
- **SSD/HDD (TB, 50000 cycles)**: 영구 저장 장치. CPU 입장에서는 극도로 느리다.

**cycle 수치의 의미**: 1 cycle은 CPU 클럭 한 틱이다. 3GHz CPU 기준 1 cycle ≈ 0.33ns. L1 접근에 2-3 cycles(~1ns)이면 되지만, Main Memory는 50-200 cycles(~17-67ns), SSD는 50000 cycles(~16μs) 이상 걸린다. 계층 간 속도 차이가 수십~수만 배에 달한다.

---

## System Bus (시스템 버스)

다이어그램은 Processor, Cache, Memory, I/O 장치가 **System Bus**를 통해 연결된 구조를 보여준다.

```
    Processor
        │
      Cache  ← Cache Hit이면 여기서 바로 반환 (System Bus 미사용)
        │
        │ Cache Miss이면 아래로 내려감
        ▼
═══ System Bus ═══════════════
        │              │
     Memory           I/O
```

**동작 흐름**:
1. Processor가 데이터를 요청하면 먼저 Processor와 System Bus 사이에 위치한 **Cache**를 확인한다.
2. **Cache Hit**: 데이터가 캐시에 있으면 바로 반환한다. System Bus를 사용하지 않으므로 빠르다.
3. **Cache Miss**: 캐시에 없으면 요청이 System Bus를 통해 **Memory**에 도달한다. Bus를 경유하므로 느리다.

**System Bus란**: CPU, 메모리, I/O 장치 간에 데이터를 전달하는 공유 통신 경로다. Address Bus(주소 전달), Data Bus(데이터 전달), Control Bus(제어 신호 전달)로 구성된다. 캐시가 존재하는 이유는 이 Bus의 대역폭이 제한적이고, Bus를 통한 메모리 접근이 느리기 때문이다.

---

## Size & Latency (크기와 지연 시간 비교)

이 다이어그램은 Jeff Dean이 발표한 유명한 **"Latency Numbers Every Programmer Should Know"** 차트다. 각 작업의 소요 시간을 시각적 블록으로 비교한다.

| 작업 | 지연 시간 | 비고 |
|------|----------|------|
| L1 cache reference | **0.5 ns** | 기준점 |
| Branch mispredict | 5 ns | CPU 파이프라인 예측 실패 |
| L2 cache reference | 7 ns | L1의 14배 |
| Mutex lock/unlock | 25 ns | 동기화 비용 |
| Main memory reference | **100 ns** | L1의 200배 |
| Compress 1KB (Zippy) | 3 μs | 3,000 ns |
| Send 1KB over 1Gbps network | 10 μs | 네트워크 전송 |
| Read 1MB sequentially from memory | 250 μs | |
| Round trip in same datacenter | 500 μs | 같은 DC 내 |
| SSD random read | 150 μs | |
| Read 1MB sequentially from SSD | 1 ms | |
| Disk seek (HDD) | 10 ms | SSD의 66배 |
| Read 1MB sequentially from disk | 20 ms | |
| Packet roundtrip CA to Netherlands | **150 ms** | 대륙 간 네트워크 |

**시사점**: L1 캐시(0.5ns)와 디스크(10ms)의 차이는 약 **2천만 배**다. 네트워크(150ms)와는 **3억 배** 차이. 이 수치가 "왜 캐시가 필수인가"에 대한 근본적 답이다. 자주 접근하는 데이터를 빠른 계층에 두는 것만으로도 시스템 성능이 극적으로 개선된다.

---

## Pareto Principle (파레토 법칙)과 캐시

### 80/20 법칙

다이어그램은 두 개의 원형 차트로 **Efforts(노력) vs Results(결과)** 의 비대칭 관계를 보여준다.

- 전체 노력의 **20%**(Important)가 전체 결과의 **80%** 를 만들어낸다.
- 나머지 80%의 노력은 20%의 결과만 만든다.

**캐시 적용**: 시스템 트래픽에서도 동일한 패턴이 나타난다. 일일 활성 트래픽의 **20%가 전체 사용 패턴의 80%** 를 차지한다. 따라서 이 상위 20% 데이터만 캐시해도 전체 요청의 80%를 캐시에서 처리할 수 있다. 캐시 용량 산정의 기본 근거가 된다.

### 머리와 꼬리 (Long Tail)

다이어그램은 X축(제품 순위), Y축(판매량)의 그래프에서 전형적인 **롱테일 분포**를 보여준다.

- **머리(Head)**: 소수의 인기 제품이 판매량 대부분을 차지 → 파레토 법칙
- **꼬리(Long Tail)**: 다수의 비인기 제품이 소량씩 판매됨

캐시 전략에서는 **머리 부분(Hot Data)** 을 캐시에 저장하는 것이 효율적이다. 꼬리 부분은 접근 빈도가 낮으므로 캐시 적중률(hit ratio)에 기여가 낮다.

---

## Cache Coherence (캐시 일관성)

멀티코어/멀티프로세서 환경에서 각 코어가 자체 캐시를 가지고 있을 때, 한 코어가 데이터를 변경하면 **다른 코어의 캐시에도 이 변경 사항이 전파**되어야 한다.

**문제 상황 예시**:
1. Core 1이 변수 X=10을 L1 캐시에 로드
2. Core 2도 같은 X=10을 자기 L1 캐시에 로드
3. Core 1이 X=20으로 변경 → Core 1의 캐시에는 X=20
4. Core 2는 여전히 X=10 → **불일치(Incoherent)**

**해결 프로토콜**:
- **MESI Protocol**: 캐시 라인의 상태를 Modified, Exclusive, Shared, Invalid 4가지로 관리. 한 코어가 데이터를 수정하면 다른 코어의 해당 캐시 라인을 Invalid로 표시한다.
- **Snooping**: 각 캐시가 Bus를 감시(snoop)하여 다른 코어의 쓰기를 감지하고 자기 캐시를 무효화한다.
- **Directory-based**: 중앙 디렉터리가 어떤 캐시가 어떤 블록을 보유하는지 추적한다.

읽기는 여러 코어가 동시에 해도 문제 없지만, **쓰기가 발생하면 일관성 유지 비용이 발생**한다. 이것이 "읽기보다 쓰기가 어렵다"는 의미다.

---

## Cache Eviction Policies (캐시 교체 정책)

캐시 용량이 가득 찼을 때, 새 데이터를 위해 **어떤 기존 데이터를 제거할지** 결정하는 알고리즘이다.

### FIFO (First In First Out)
- **구조**: Queue
- **동작**: 가장 먼저 캐시에 들어온 항목을 먼저 제거
- **장점**: 구현이 단순
- **단점**: 자주 사용되는 데이터도 오래 전에 들어왔다면 제거됨. 접근 빈도를 고려하지 않음

### LFU (Least Frequently Used)
- **구조**: 각 항목에 접근 카운터 부착
- **동작**: 사용 횟수(frequency)가 가장 적은 항목을 제거
- **장점**: 인기 데이터를 오래 보존
- **단점**: 과거에 많이 사용됐지만 이제 불필요한 데이터가 계속 남음 (frequency pollution). 카운터 관리 오버헤드

### LRU (Least Recently Used)
- **구조**: Doubly Linked List + HashMap
- **동작**: 가장 오랫동안 접근하지 않은(가장 오래된) 항목을 제거
- **장점**: 최근 접근 패턴을 잘 반영. 시간 지역성과 잘 맞음
- **단점**: 갑자기 대량의 새 데이터가 유입되면 기존 Hot 데이터가 밀려남 (cache pollution)
- **구현 원리**: 접근할 때마다 해당 노드를 리스트 맨 앞(head)으로 이동. 제거 시 맨 뒤(tail) 노드를 삭제

### Windowed-LFU
- **동작**: **최근 일정 시간 창(window)** 내의 사용 빈도를 기준으로 판단
- **LRU와의 차이**: LRU는 마지막 접근 시점만 보지만, W-LFU는 최근 window 내 빈도를 본다
- **Caffeine 라이브러리**: Java의 Caffeine 캐시가 W-TinyLFU 알고리즘을 사용하여 LRU보다 높은 적중률을 달성

---

## Cache Strategies — Read (읽기 전략)

### Cache Aside (Lazy Loading)

다이어그램의 흐름:
```
App ──1:READ──▶ Cache
               2:MISS (캐시에 없음)
App ──3:READ ORIGINAL──▶ DB
App ◀──4:GET──────────── DB
App ──5:UPDATE──▶ Cache (DB에서 읽은 데이터를 캐시에 저장)
```

**동작 원리**:
1. 애플리케이션이 **캐시에 먼저** 데이터를 조회한다.
2. **Hit**: 캐시에서 바로 반환.
3. **Miss**: 애플리케이션이 **직접 DB에 쿼리**하고, 받아온 데이터를 **캐시에 저장**한 뒤 반환한다.

**특징**:
- **애플리케이션이 캐시와 DB 모두를 직접 관리**한다 (캐시가 자동으로 DB를 읽지 않음).
- 최초 요청이나 캐시 만료 후에는 반드시 Miss가 발생하여 **latency가 증가**한다.
- **캐시 장애 시에도 DB에서 직접 조회 가능** → 전체 서비스 장애로 전파되지 않음.
- **동기화 문제**: DB가 갱신되어도 캐시가 자동 갱신되지 않아 stale 데이터가 남을 수 있다. TTL이나 명시적 invalidation 필요.
- **Redis를 이용한 대부분의 캐시**로 많이 사용한다.

### Read Through

다이어그램의 흐름:
```
App ──1:READ──▶ Cache
               2:MISS
               Cache ──3:READ ORIGINAL──▶ DB
               Cache ◀──4:GET──────────── DB
               5:UPDATE (캐시가 스스로 DB 데이터를 캐시에 저장)
App ◀────────── Cache (캐시가 데이터 반환)
```

**동작 원리**:
1. 애플리케이션은 **캐시에만** 데이터를 요청한다.
2. **Hit**: 캐시에서 바로 반환.
3. **Miss**: **캐시가 스스로 DB에서 데이터를 가져와** 자기 자신을 갱신한 뒤, 애플리케이션에 반환한다.

**Cache Aside와의 핵심 차이**:
- Cache Aside: **애플리케이션**이 DB를 직접 조회하고 캐시에 넣는다.
- Read Through: **캐시 계층**이 DB 조회와 자기 갱신을 자동으로 처리한다. 애플리케이션은 캐시만 알면 된다.

**특징**:
- 캐시 스키마 == 원본 스키마 (캐시가 중간 변환 없이 원본을 그대로 저장)
- 최초 요청 시 **반드시 Cache Miss** 발생 (warm-up 필요)
- 캐시 크기 == 원본 크기 (캐시에 저장되는 데이터 형태가 원본과 동일)
- **CDN, Reverse Proxy**가 대표적 Read Through 구현체. 클라이언트는 CDN에만 요청하고, CDN이 Miss 시 Origin 서버에서 가져온다.