---
title: "K8S는 JVM 앱을 어떻게 살리고 죽이는가 — Probe부터 OOM Kill까지"
date: 2026-05-08 14:00:00 +0900
categories: [JVM, Kubernetes]
tags: [kubernetes, jvm, spring-boot, probe, graceful-shutdown, hpa, jit-warmup, oom, monitoring, lifecycle]
---

> **TL;DR**: K8S는 JVM 앱을 배포하고, Probe로 생사를 판단하고, 트래픽을 제어하고, 종료 시 정리할 시간을 준다. 이 과정에서 JVM 특유의 느린 시작, 좀비 상태, Warmup 문제를 이해하지 못하면 장애로 이어진다.

---

## 1. 클러스터 구조

```text
┌────────────────────────── Kubernetes Cluster ──────────────────────────┐
│                                                                        │
│  ┌─── Control Plane ───┐     ┌──────── Worker Node ────────────────┐  │
│  │                      │     │                                     │  │
│  │  API Server          │     │  kubelet: Pod 관리, Probe 실행      │  │
│  │  Scheduler           │     │  kube-proxy: Service → Pod 라우팅   │  │
│  │  Controller Manager  │     │  Container Runtime: 컨테이너 실행   │  │
│  │  etcd                │     │                                     │  │
│  └──────────────────────┘     │  ┌─Pod─┐  ┌─Pod─┐  ┌─Pod─┐        │  │
│                                │  │ JVM │  │ JVM │  │ JVM │        │  │
│                                │  └─────┘  └─────┘  └─────┘        │  │
│                                └─────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

- **API Server**: 모든 클러스터 조작의 진입점
- **Scheduler**: Pod를 어떤 Node에 배치할지 결정 (리소스 요구량, affinity, taint 등 고려)
- **kubelet**: Node에서 Pod를 관리하고 컨테이너 상태(Probe)를 확인
- **kube-proxy**: Service의 가상 IP를 실제 Pod IP로 라우팅 (iptables/IPVS 규칙 관리)

> **근거**: Kubernetes 공식 문서 — *"Control Plane은 클러스터에 대한 전역적 결정을 내리고, 각 Worker Node의 kubelet이 Pod의 컨테이너를 관리한다."* ([Kubernetes - Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/))

---

## 2. JVM 앱의 라이프사이클

K8S가 JVM 앱을 **어떻게 살리는지** — 배포 요청부터 트래픽 유입까지의 전체 과정이다.

```text
① 배포 요청
   kubectl apply / CI-CD pipeline
         │
         ▼
② API Server → etcd 저장
         │
         ▼
③ Deployment Controller → ReplicaSet 생성 → Pod 생성 (Pending 상태)
         │
         ▼
④ Scheduler: Filtering → Scoring → Node 선택 → Binding
   Filtering: 리소스 충족 여부, Taint/Toleration, NodeAffinity 확인
   Scoring:   이미지 캐시 여부, CPU/메모리 균형 등으로 최적 Node 선정
         │
         ▼
⑤ kubelet: 이미지 Pull → 컨테이너 시작 → ENTRYPOINT 실행
         │
         ▼
⑥ JVM 부팅 (이 단계가 느림)
   클래스 로딩 → Metaspace 할당 → JIT 컴파일러 초기화
   Spring Context 생성 → Bean 스캔 → DI → 커넥션 풀 수립
         │
         ▼
⑦ Probe 통과
   Startup Probe 성공 → Liveness/Readiness Probe 시작
   Readiness 성공 → Endpoint에 Pod IP 추가 → 트래픽 유입 시작
         │
         ▼
⑧ 정상 운영 (서비스)
         │
         ▼
⑨ 종료 (Section 4. Graceful Shutdown 참조)
```

> **근거**: Kubernetes 공식 문서 — Scheduler는 2단계(Filtering, Scoring) 과정으로 Pod를 Node에 배치한다. Filtering에서 리소스 요구사항을 충족하지 못하는 Node를 제외하고, Scoring에서 최적의 Node를 선택한다. ([Kubernetes - kube-scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/))

---

## 3. Health Check — 세 가지 Probe

JVM 앱은 시작이 느리기 때문에 Probe 설정이 중요하다. 각 Probe가 **없으면 어떤 문제가 생기는지** 시나리오로 이해하면 역할이 명확해진다.

### Startup Probe — "아직 시작 중인가?"

Startup Probe가 성공할 때까지 Liveness/Readiness Probe는 **비활성화 상태**를 유지한다.

```text
Startup Probe가 없을 때:

  Pod 시작 → Spring Context 초기화 중 (30초 소요)
                │
                ├── Liveness Probe: /actuator/health → 응답 없음 → 실패 3회
                │
                ▼
          kubelet: "프로세스가 죽었구나" → 컨테이너 재시작
                │
                ▼
          또 30초 초기화 → 또 Liveness 실패 → 또 재시작
                │
                ▼
          CrashLoopBackOff (무한 재시작 루프)

Startup Probe가 있을 때:

  Pod 시작 → Spring Context 초기화 중 (30초 소요)
                │
                ├── Startup Probe: 실패해도 OK (failureThreshold 36 = 최대 190초 대기)
                ├── Liveness/Readiness: 아직 비활성화 (Startup 통과 전까지)
                │
                ▼ (30초 후)
          Startup Probe 성공 → Liveness/Readiness 활성화 시작
```

### Liveness Probe — "프로세스가 응답하는가?"

실패 시 kubelet이 컨테이너를 **재시작**한다. 프로세스가 살아있지만 정상 동작하지 않는 **좀비 상태**를 감지하는 것이 목적이다.

```text
JVM OOM 후 좀비 상태 — Liveness Probe가 없을 때:

  Client ──요청──▶ Pod (좀비)
                    │
                    ├── 프로세스: 살아있음 (PID 존재)
                    ├── Heap: 꽉 참 (OutOfMemoryError 발생)
                    ├── 새 요청마다: 500 Internal Server Error
                    │
                    └── Kubernetes: Pod 상태 Running → "정상이니 놔두자"
                        → 영원히 에러만 반환하는 좀비 Pod

Liveness Probe가 있을 때:

  Client ──요청──▶ Pod (좀비)
                    │
                    ├── Liveness Probe: /actuator/health → 200 OK (Actuator는 응답 가능)
                    │   ⚠ Actuator health check는 통과하지만 비즈니스 요청은 실패
                    │   → 이것이 ExitOnOutOfMemoryError가 필요한 이유
                    │     (JVM OOM 시 프로세스를 종료시켜야 Liveness도 실패)
                    │
                    ├── ExitOnOutOfMemoryError로 JVM 종료 → Liveness 실패
                    │
                    ▼
                kubelet: 컨테이너 재시작 → 정상 복구
```

### Readiness Probe — "트래픽 받을 준비 되었나?"

실패 시 Service의 Endpoint 목록에서 **제거**된다. 재시작이 아니라 **트래픽만 끊는다**. 일시적으로 요청을 처리할 수 없는 상황에서 다른 정상 Pod로 트래픽을 우회시키는 것이 목적이다.

```text
DB 장애 시 — Readiness Probe가 없을 때:

  ┌── Service (kube-proxy) ──────────────────────────────────┐
  │                                                           │
  │  트래픽 분배:                                             │
  │    Pod A (정상)  ← 33%                                   │
  │    Pod B (DB 연결 끊김) ← 33% → 전부 500 에러 반환       │
  │    Pod C (정상)  ← 33%                                   │
  │                                                           │
  │  결과: 전체 요청의 33%가 실패                             │
  └───────────────────────────────────────────────────────────┘

Readiness Probe가 있을 때:

  Pod B: Readiness Probe(/actuator/health/readiness)
         → DB health check 포함 → DB 연결 끊김 → 실패
         → Endpoint 목록에서 Pod B 제거

  ┌── Service (kube-proxy) ──────────────────────────────────┐
  │                                                           │
  │  트래픽 분배:                                             │
  │    Pod A (정상)  ← 50%                                   │
  │    Pod B (제거됨, 트래픽 안 옴, 재시작도 안 함)           │
  │    Pod C (정상)  ← 50%                                   │
  │                                                           │
  │  결과: 정상 Pod만 트래픽 수신, 에러 0%                    │
  │  DB 복구되면 → Readiness 성공 → 자동으로 Endpoint 복귀   │
  └───────────────────────────────────────────────────────────┘
```

### Liveness vs Readiness를 잘못 쓰면?

```text
잘못된 설정: Liveness Probe에 DB health check를 넣은 경우

  DB 일시적 장애 (30초간)
    → 모든 Pod의 Liveness Probe 실패
      → kubelet이 모든 Pod 동시 재시작
        → 재시작 중 트래픽 처리 불가 (전체 서비스 다운)
          → DB 복구되어도 Pod 재시작 완료까지 수십 초 추가 장애

올바른 설정: Readiness에만 DB check, Liveness는 가볍게

  DB 일시적 장애 (30초간)
    → 모든 Pod의 Readiness 실패 → Endpoint에서 제거 (트래픽 차단)
    → Liveness는 통과 → 재시작 안 함 (Pod 살아있음)
    → DB 복구 → Readiness 성공 → Endpoint 복귀 → 즉시 정상화
```

### Probe 설정 정리

| Probe | 실패 시 동작 | 핵심 역할 | 외부 의존성 포함? |
|---|---|---|---|
| **Startup** | Liveness/Readiness 비활성화 유지 | JVM 느린 시작 수용 | 불필요 |
| **Liveness** | 컨테이너 **재시작** | 좀비 Pod 감지 (+ ExitOnOOM 병행) | **넣지 않는다** |
| **Readiness** | Endpoint에서 **제거** (재시작 아님) | 일시적 장애 시 트래픽 우회 | **넣는다** (DB, 캐시) |

```yaml
# Spring Boot Actuator 연동 예시
startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 36     # 최대 10 + (5×36) = 190초 대기

livenessProbe:
  httpGet:
    path: /actuator/health/liveness    # 가벼운 체크만 (DB 체크 X)
    port: 8080
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness   # DB, 캐시 등 외부 의존성 포함
    port: 8080
  periodSeconds: 5
  failureThreshold: 3
```

> **근거**: Kubernetes 공식 문서 — *"Startup Probe가 성공할 때까지 Liveness, Readiness Probe는 비활성화된다."* *"Readiness Probe 실패 시 Endpoint Controller가 해당 Pod의 IP를 모든 Service의 Endpoint에서 제거한다."* Spring Blog — *"Spring Boot 2.3+에서 `/actuator/health/liveness`(가벼운 내부 상태), `/actuator/health/readiness`(외부 의존성 포함) 엔드포인트를 분리 제공한다."* ([Kubernetes - Configure Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/), [Spring.io - Liveness and Readiness Probes](https://spring.io/blog/2020/03/25/liveness-and-readiness-probes-with-spring-boot/))

---

## 4. Graceful Shutdown — K8S는 JVM 앱을 어떻게 죽이는가

Pod 삭제 시 다음이 **병렬로** 시작된다:

```text
┌─ Track A: 네트워크 ──────────────────┐  ┌─ Track B: 컨테이너 종료 ─────────────────────┐
│                                       │  │                                               │
│  Endpoint에서 Pod 제거                │  │  terminationGracePeriodSeconds 카운트다운 시작 │
│          │                            │  │          │                                    │
│          ▼                            │  │          ▼                                    │
│  kube-proxy 규칙 업데이트 (수초 소요) │  │  preStop hook 실행 (예: sleep 10)             │
│          │                            │  │          │                                    │
│          ▼                            │  │          ▼                                    │
│  새 요청이 이 Pod로 오지 않음         │  │  SIGTERM → JVM Shutdown Hook 실행             │
│                                       │  │          │                                    │
│                                       │  │          ▼                                    │
│                                       │  │  Spring: 새 요청 거부 + 진행 중 요청 완료     │
│                                       │  │  @PreDestroy: 커넥션 풀 반환, 리소스 정리     │
│                                       │  │          │                                    │
│                                       │  │          ▼                                    │
│                                       │  │  JVM 정상 종료                                │
│                                       │  │  (유예 초과 시 SIGKILL 강제 종료)             │
└───────────────────────────────────────┘  └───────────────────────────────────────────────┘
```

### Race Condition

Track A(규칙 업데이트)보다 Track B(앱 종료)가 먼저 끝나면, kube-proxy가 아직 규칙을 안 바꿔서 이미 죽은 Pod로 요청이 간다. → **`preStop: sleep 10`**으로 해결.

```text
타이밍 공식:
  terminationGracePeriodSeconds >= preStop(10s) + Spring shutdown(25s) + 여유(5s) = 40s 이상
  기본값 30초가 위험한 이유: preStop(10s) + Spring(30s) = 40s > 30s → SIGKILL로 강제 종료됨
```

### Spring Boot Graceful Shutdown 설정

```yaml
# application.yml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s   # terminationGracePeriodSeconds보다 작아야 함

# Deployment
spec:
  terminationGracePeriodSeconds: 45   # preStop(10) + Spring(25) + 여유(10)
  containers:
  - lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "sleep 10"]
```

> **근거**: Google Cloud Blog — *"Pod 종료 시 SIGTERM과 Endpoint 제거는 동시에 시작된다. preStop hook으로 앱이 Endpoint 제거보다 먼저 죽는 것을 방지해야 한다."* CNCF Blog — *"terminationGracePeriodSeconds는 전체 종료 과정(preStop + SIGTERM 처리)을 포함하는 유예 시간이다."* ([Google Cloud - Terminating with grace](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-terminating-with-grace), [CNCF - Pod termination lifecycle](https://www.cncf.io/blog/2024/12/19/decoding-the-pod-termination-lifecycle-in-kubernetes-a-comprehensive-guide/))

---

## 5. 모니터링 — JVM Metrics

```text
Spring Boot App
  └─ Actuator + Micrometer → /actuator/prometheus 엔드포인트
                                       │
                                Prometheus (수집/저장)
                                       │
                                Grafana (시각화)
```

JVM 앱에서 필수로 관찰해야 하는 메트릭:

| 메트릭 | 설명 | 왜 중요한가 |
|---|---|---|
| `jvm_memory_used_bytes{area="heap"}` | Heap 사용량 | Pod OOM 예측 |
| `jvm_memory_used_bytes{area="nonheap"}` | Metaspace + CodeCache | Metaspace 누수 감지 |
| `jvm_gc_pause_seconds` | GC Pause 시간 | 지연시간 영향 파악 |
| `jvm_threads_live_threads` | 활성 스레드 수 | 스레드 누수 감지 (Thread Stack 메모리 증가) |

> **근거**: Spring Boot 공식 문서 — Micrometer가 자동으로 JVM 메트릭(Heap, Non-Heap, GC, Thread 등)을 수집하며, `micrometer-registry-prometheus` 의존성 추가만으로 Prometheus 형식으로 노출된다. ([Spring Boot - Metrics](https://docs.spring.io/spring-boot/reference/actuator/metrics.html))

---

## 6. HPA와 JVM Warmup 문제

HPA(Horizontal Pod Autoscaler)는 메트릭 기반으로 Pod 수를 자동 조절한다. 그런데 JVM 앱에서는 **Cold Start 문제**가 있다.

```text
트래픽 스파이크
  → HPA가 새 Pod 생성
    → 새 Pod의 JVM은 JIT 미컴파일 상태 → CPU 급등
      → HPA가 이를 "부하 증가"로 오인 → 추가 Pod 생성
        → 새 Pod도 차가운 상태 → CPU 급등
          → 스케일링 폭주 (Scaling Storm)
```

**원인**: JVM은 JIT(Just-In-Time) 컴파일러를 사용한다. 시작 직후에는 인터프리터 모드로 실행하다가 자주 호출되는 코드를 점진적으로 네이티브 코드로 컴파일한다. 이 "워밍업" 기간에 CPU를 많이 소모한다.

**대응 전략**:
1. **Application-Level Warmup**: 시작 시 주요 코드 경로를 미리 호출하여 JIT 유도. Readiness Probe를 warmup 완료 후에만 성공시킴.
2. **HPA 튜닝**: `stabilizationWindowSeconds: 120`으로 스케일 업 판단 전 2분 안정화 대기, 한 번에 최대 2개만 추가.
3. **CRaC (Coordinated Restore at Checkpoint)**: 워밍업된 JVM 스냅샷을 저장/복원하여 시작 시간을 수초로 단축.

> **근거**: BlaBlaCar Engineering — *"JIT 컴파일이 안 된 새 Pod는 CPU를 과도하게 소모하여 HPA가 추가 스케일 아웃을 유발한다."* Kubernetes HPA 문서 — *"cpuInitializationPeriod(기본 5분) 동안 unready Pod의 CPU 사용량을 무시하는 보호 메커니즘이 있다."* ([BlaBlaCar - Java and Kubernetes warmup](https://medium.com/blablacar/warm-up-the-relationship-between-java-and-kubernetes-7fc5741f9a23), [Kubernetes - HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/))

