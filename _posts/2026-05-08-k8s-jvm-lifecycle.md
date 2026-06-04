---
title: "K8S에서 JVM 앱 운영하기 (1) — 클러스터 구조와 Probe"
date: 2026-05-08 10:00:00 +0900
categories: [JVM, Kubernetes]
tags: [kubernetes, jvm, probe, startup-probe, liveness-probe, readiness-probe, lifecycle]
series: "K8S에서 JVM 앱 운영하기"
---

> 이 글은 **K8S에서 JVM 앱 운영하기** 시리즈의 첫 번째 글이다.
> 1. **클러스터 구조와 Probe** ← 현재 글
> 2. [Graceful Shutdown과 HPA Warmup](/posts/k8s-jvm-operations/)
> 3. [CPU Throttling과 메모리 한계 동작](/posts/k8s-jvm-memory/)

CPU/Memory 동작 차이와 OOM 문제를 이해하려면, K8S가 JVM 앱을 배포·실행·종료하는 전체 라이프사이클을 먼저 파악해야 한다. 이 글에서는 K8S 클러스터 구조, JVM 앱의 배포 흐름, 그리고 JVM 앱에서 특히 중요한 세 가지 Probe를 다룬다.

---

## 1. 클러스터 구조

```text
┌────────────────────────── Kubernetes Cluster ──────────────────────────┐
│                                                                        │
│  ┌─── Control Plane ───┐     ┌──────── Worker Node ────────────────┐   │
│  │                      │     │                                     │  │
│  │  API Server          │     │  kubelet: Pod 관리, Probe 실행      │  │
│  │  Scheduler           │     │  kube-proxy: Service → Pod 라우팅   │  │
│  │  Controller Manager  │     │  Container Runtime: 컨테이너 실행   │  │
│  │  etcd                │     │                                     │  │
│  └──────────────────────┘     │  ┌─Pod─┐  ┌─Pod─┐  ┌─Pod─┐        │    │
│                                │  │ JVM │  │ JVM │  │ JVM │        │   │
│                                │  └─────┘  └─────┘  └─────┘        │   │
│                                └─────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

- **API Server**: 모든 클러스터 조작의 진입점
- **Scheduler**: Pod를 어떤 Node에 배치할지 결정 (리소스 요구량, affinity, taint 등 고려)
- **kubelet**: Node에서 Pod를 관리하고 컨테이너 상태(Probe)를 확인
- **kube-proxy**: Service의 가상 IP를 실제 Pod IP로 라우팅 (iptables/IPVS 규칙 관리)

> **근거**: Kubernetes 공식 문서 — *"Control Plane은 클러스터에 대한 전역적 결정을 내리고, 각 Worker Node의 kubelet이 Pod의 컨테이너를 관리한다."* ([Kubernetes - Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/))

---

## 2. JVM 앱의 라이프사이클

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
⑨ 종료 (시리즈 2편 Graceful Shutdown 참조)
```

> **근거**: Kubernetes 공식 문서 — Scheduler는 2단계(Filtering, Scoring) 과정으로 Pod를 Node에 배치한다. Filtering에서 리소스 요구사항을 충족하지 못하는 Node를 제외하고, Scoring에서 최적의 Node를 선택한다. ([Kubernetes - kube-scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/))

---

## 3. Health Check — 세 가지 Probe

JVM 앱은 클래스 로딩과 Spring Context 초기화로 기동 시간이 수십 초에 달하므로 Probe 설정이 중요하다. 각 Probe가 없을 때 발생하는 장애 시나리오를 통해 역할을 구분한다.

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
          CrashLoopBackOff (재시작 백오프)

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
                    ├── 새 요청: 실패하거나 지연될 수 있음
                    │
                    └── Kubernetes: Pod 상태 Running → "정상이니 놔두자"
                        → 계속 에러를 반환하는 Pod가 남을 수 있음

Liveness Probe가 있을 때:

  Client ──요청──▶ Pod (좀비)
                    │
                    ├── Liveness Probe: /actuator/health → 200 OK일 수 있음
                    │   ⚠ health check는 통과하지만 비즈니스 요청은 실패할 수 있음
                    │   → 이것이 ExitOnOutOfMemoryError가 필요한 이유
                    │     (JVM OOM 시 프로세스를 종료시키면 kubelet이 재시작 가능)
                    │
                    ├── ExitOnOutOfMemoryError 설정 시 JVM 종료 → 컨테이너 종료
                    │
                    ▼
                kubelet: restartPolicy에 따라 컨테이너 재시작
```

### Readiness Probe — "트래픽 받을 준비 되었나?"

실패 시 Service의 Endpoint 목록에서 **제거**된다. 재시작이 아니라 **트래픽만 끊는다**. 일시적으로 요청을 처리할 수 없는 상황에서 다른 정상 Pod로 트래픽을 우회시키는 것이 목적이다.

```text
DB 장애 시 — Readiness Probe가 없을 때:

  ┌── Service (kube-proxy) ───────────────────────────────────┐
  │                                                           │
  │  트래픽 분배:                                             │
  │    Pod A (정상)  ← 33%                                    │
  │    Pod B (DB 연결 끊김) ← 33% → 관련 요청 실패 가능       │
  │    Pod C (정상)  ← 33%                                    │
  │                                                           │
  │  결과: 일부 요청이 실패할 수 있음                         │
  └───────────────────────────────────────────────────────────┘

Readiness Probe가 있을 때:

  Pod B: Readiness Probe(/actuator/health/readiness)
         → DB health check 포함 → DB 연결 끊김 → 실패
         → Endpoint 목록에서 Pod B 제거

  ┌── Service (kube-proxy) ───────────────────────────────────┐
  │                                                           │
  │  트래픽 분배:                                             │
  │    Pod A (정상)  ← 50%                                    │
  │    Pod B (제거됨, 트래픽 안 옴, 재시작도 안 함)           │
  │    Pod C (정상)  ← 50%                                    │
  │                                                           │
  │  결과: 준비된 Pod만 트래픽 수신                           │
  │  DB 복구되면 → Readiness 성공 → 자동으로 Endpoint 복귀    │
  └───────────────────────────────────────────────────────────┘
```

단, Readiness에 외부 의존성을 넣을지는 신중히 판단해야 한다. 모든 Pod가 같은 DB나 캐시에 의존하는 구조에서 해당 의존성이 장애 나면, 모든 Pod가 동시에 Unready가 되어 Service 관점에서는 가용 엔드포인트가 사라질 수 있다. Spring Boot도 liveness에는 외부 시스템 health check를 넣지 말라고 권고하고, readiness의 외부 의존성 포함 여부는 애플리케이션의 장애 처리 전략에 맞게 결정하라고 설명한다.

**Liveness vs Readiness를 잘못 쓰면?**

```text
잘못된 설정: Liveness Probe에 DB health check를 넣은 경우

  DB 일시적 장애 (30초간)
    → 모든 Pod의 Liveness Probe 실패
      → kubelet이 모든 Pod 동시 재시작
        → 재시작 중 가용 Pod 감소
          → DB 복구 이후에도 Pod 재시작 시간만큼 복구 지연 가능

권장 설정: Liveness는 내부 생존 신호 위주, Readiness는 필요 시 DB check 포함

  DB 일시적 장애 (30초간)
    → 모든 Pod의 Readiness 실패 → Endpoint에서 제거 (트래픽 차단)
    → Liveness는 통과 → 재시작 안 함 (Pod 살아있음)
    → DB 복구 → Readiness 성공 → Endpoint 복귀
```

### Probe 설정 정리

| Probe | 실패 시 동작 | 핵심 역할 | 외부 의존성 포함? |
|---|---|---|---|
| **Startup** | Liveness/Readiness 비활성화 유지 | JVM 느린 시작 수용 | 불필요 |
| **Liveness** | 컨테이너 **재시작** | 좀비 Pod 감지 (+ ExitOnOOM 병행) | **넣지 않는다** |
| **Readiness** | Endpoint에서 **제거** (재시작 아님) | 일시적 장애 시 트래픽 우회 | 필요 시 포함하되, 공유 의존성은 신중히 판단 |

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
    path: /actuator/health/readiness   # 필요 시 DB, 캐시 등 외부 의존성 포함
    port: 8080
  periodSeconds: 5
  failureThreshold: 3
```

> **근거**: Kubernetes 공식 문서 — Startup Probe가 설정되면 성공 전까지 Liveness/Readiness Probe를 실행하지 않는다. Readiness Probe가 실패하면 Pod의 Ready condition이 false가 되어 Service backend에서 제외된다. Spring Boot 문서는 liveness에는 외부 시스템 health check를 넣지 말고, readiness의 외부 의존성 포함 여부는 애플리케이션 의도에 맞게 신중히 결정하라고 설명한다. ([Kubernetes - Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/), [Spring Boot - Kubernetes Probes](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html#actuator.endpoints.kubernetes-probes))

---

다음 글에서는 Pod 종료 시 발생하는 **Graceful Shutdown Race Condition**, JVM 모니터링, HPA와 JVM Warmup 문제를 다룬다.
