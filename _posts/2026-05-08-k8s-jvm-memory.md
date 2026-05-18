---
title: "K8S에서 JVM 앱 운영하기 — CPU는 느려지고, Memory는 죽는다"
date: 2026-05-08 10:00:00 +0900
categories: [JVM, Kubernetes]
tags: [kubernetes, jvm, oom, pod-resources, memory, cpu-throttling, probe, graceful-shutdown, hpa, monitoring, qos]
---

> **TL;DR**: K8S에서 JVM 앱을 운영할 때 CPU limit 초과와 Memory limit 초과는 커널의 처리 방식이 다르다. CPU limit 초과는 CFS throttling으로 제한되고, memory limit 초과는 커널 OOM Killer가 SIGKILL을 전송해 프로세스를 강제 종료할 수 있다. JVM OOM과 Pod OOM은 감지 주체, 증상, 대응 방법이 모두 다르다.

```text
┌─────────────────────────────────────────────────────────┐
│             Pod가 리소스 limit을 초과하면?              │
├────────────────────────────┬────────────────────────────┤
│                            │                            │
│          CPU 초과          │         Memory 초과        │
│       (compressible)       │      (incompressible)      │
│                            │                            │
│             │              │             │              │
│             v              │             v              │
│   ┌────────────────────┐   │   ┌────────────────────┐   │
│   │  CFS Throttling    │   │   │  OOM Killer        │   │
│   │  커널 스케줄러가   │   │   │  커널이 SIGKILL    │   │
│   │  CPU 시간을 제한   │   │   │  (signal 9) 전송   │   │
│   └─────────┬──────────┘   │   └─────────┬──────────┘   │
│             │              │             │              │
│             v              │             v              │
│   ┌────────────────────┐   │   ┌────────────────────┐   │
│   │  Pod: 살아있음     │   │   │  Pod: 종료 가능    │   │
│   │  응답만 느려짐     │   │   │  exit code 137     │   │
│   └────────────────────┘   │   └────────────────────┘   │
│                            │                            │
├────────────────────────────┴────────────────────────────┤
│             CPU는 느려지고, Memory는 죽는다             │
└─────────────────────────────────────────────────────────┘
```

```text
┌─────────────────────────────────────────────────────────┐
│          Memory 초과 시, 누가 먼저 감지하느냐?          │
├────────────────────────────┬────────────────────────────┤
│                            │                            │
│     JVM 내부 영역 초과     │ cgroup memory > limit      │
│    (Heap, Metaspace 등)    │                            │
│                            │                            │
│             │              │             │              │
│             v              │             v              │
│   ┌────────────────────┐   │   ┌────────────────────┐   │
│   │  JVM OOM           │   │   │  Pod OOM           │   │
│   │                    │   │   │                    │   │
│   │  JVM이 스스로      │   │   │  Linux 커널이      │   │
│   │  OutOfMemoryError  │   │   │  SIGKILL 전송      │   │
│   │  를 던짐           │   │   │  (JVM 모르게)      │   │
│   └─────────┬──────────┘   │   └─────────┬──────────┘   │
│             │              │             │              │
│             v              │             v              │
│   ┌────────────────────┐   │   ┌────────────────────┐   │
│   │  힙덤프: 옵션 설정 │   │   │  힙덤프 생성: X    │   │
│   │  로그 기록: 가능   │   │   │  JVM 로그: X       │   │
│   │  프로세스: 계속    │   │   │  프로세스: 강제종료│   │
│   │  살 수도 있음      │   │   │  (JVM 덤프 불가)   │   │
│   └─────────┬──────────┘   │   └─────────┬──────────┘   │
│             │              │             │              │
│             v              │             v              │
│   ┌────────────────────┐   │   ┌────────────────────┐   │
│   │  대응 방법         │   │   │  대응 방법         │   │
│   │                    │   │   │                    │   │
│   │  - HeapDumpOnOOM   │   │   │  - JVM 옵션 조정   │   │
│   │  - 메모리 누수     │   │   │  - 컨테이너 전체   │   │
│   │    추적            │   │   │    메모리 산정     │   │
│   │  - ExitOnOOM       │   │   │  - limit에 여유    │   │
│   │    (좀비 방지)     │   │   │    확보            │   │
│   └────────────────────┘   │   └────────────────────┘   │
│                            │                            │
└────────────────────────────┴────────────────────────────┘
```

> **근거**: Kubernetes 공식 문서는 CPU limit을 커널이 강제하는 hard limit으로 설명하고, CPU 사용량은 throttling으로 제한된다고 설명한다. Memory limit은 OOM kill로 강제되지만, 커널이 메모리 압박을 감지할 때 반응적으로 적용되므로 초과 즉시 항상 종료되는 것은 아니다. JVM OOM과 Pod OOM의 차이는 예외를 던지는 주체(JVM vs 커널)가 다르기 때문에 발생한다. 여기서 Pod OOM의 기준은 JVM heap만이 아니라 컨테이너 cgroup의 전체 메모리 사용량이다. JVM OOM에서 힙덤프를 남기려면 `-XX:+HeapDumpOnOutOfMemoryError` 같은 옵션이 필요하고, Pod OOM은 JVM이 아니라 커널이 프로세스를 죽이므로 JVM 힙덤프나 shutdown hook을 기대하기 어렵다. ([Kubernetes Docs - Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/), [Oracle Java Options](https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html#advanced-runtime-options-for-java), [HeapHero - OOMKilled vs Java OOM](https://blog.heaphero.io/oomkilled-vs-java-oom-kubernetes/))

---

## 1. K8S에서 JVM 앱은 어떻게 운영되는가

CPU/Memory 동작 차이와 OOM 문제를 이해하려면, K8S가 JVM 앱을 배포·실행·종료하는 전체 라이프사이클을 먼저 파악해야 한다.

### 클러스터 구조

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

### JVM 앱의 라이프사이클

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
⑨ 종료 (아래 Graceful Shutdown 참조)
```

> **근거**: Kubernetes 공식 문서 — Scheduler는 2단계(Filtering, Scoring) 과정으로 Pod를 Node에 배치한다. Filtering에서 리소스 요구사항을 충족하지 못하는 Node를 제외하고, Scoring에서 최적의 Node를 선택한다. ([Kubernetes - kube-scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/))

### Health Check — 세 가지 Probe

JVM 앱은 클래스 로딩과 Spring Context 초기화로 기동 시간이 수십 초에 달하므로 Probe 설정이 중요하다. 각 Probe가 없을 때 발생하는 장애 시나리오를 통해 역할을 구분한다.

#### Startup Probe — "아직 시작 중인가?"

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

#### Liveness Probe — "프로세스가 응답하는가?"

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

#### Readiness Probe — "트래픽 받을 준비 되었나?"

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

#### Probe 설정 정리

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

### Graceful Shutdown — 종료 시퀀스

Pod 삭제 시 다음이 **병렬로** 시작된다:

```text
┌─ Track A: 네트워크 ──────────────────┐  ┌─ Track B: 컨테이너 종료 ───────────────────────┐
│                                       │  │                                               │
│  Endpoint에서 Pod 제거                │  │  terminationGracePeriodSeconds 카운트다운 시작│
│          │                            │  │          │                                    │
│          ▼                            │  │          ▼                                    │
│  kube-proxy 규칙 업데이트 (수초 소요) │  │  preStop hook 실행 (예: sleep 10)             │
│          │                            │  │          │                                    │
│          ▼                            │  │          ▼                                    │
│  새 요청이 이 Pod로 오지 않음         │  │  SIGTERM → JVM Shutdown Hook 실행             │
│                                       │  │          │                                    │
│                                       │  │          ▼                                    │
│                                       │  │  Spring: 새 요청 거부 + 진행 중 요청 완료     │
│                                       │  │  (graceful shutdown 설정 시)                  │
│                                       │  │  @PreDestroy: 커넥션 풀 반환, 리소스 정리     │
│                                       │  │          │                                    │
│                                       │  │          ▼                                    │
│                                       │  │  JVM 정상 종료                                │
│                                       │  │  (유예 초과 시 SIGKILL 강제 종료)             │
└───────────────────────────────────────┘  └───────────────────────────────────────────────┘
```

**Race Condition**: Track A(Endpoint/라우팅 반영)보다 Track B(앱 종료)가 먼저 끝나면, 일부 경로에서 종료 중인 Pod로 요청이 갈 수 있다. `preStop: sleep 10` 같은 지연은 이 경합 윈도우를 축소하는 완화책이며, 실제 값은 Ingress/LB/Service 전파 지연과 앱 종료 시간을 기준으로 산정해야 한다.

Spring Boot에서 진행 중 요청을 기다리는 종료를 기대하려면 애플리케이션 쪽 graceful shutdown도 켜야 한다. 예를 들어 `server.shutdown=graceful`과 `spring.lifecycle.timeout-per-shutdown-phase`를 termination grace period 안에 들어오도록 맞춘다. SIGTERM을 받는다고 모든 Spring Boot 애플리케이션이 자동으로 진행 중 요청을 끝까지 기다리는 것은 아니다.

```text
타이밍 공식:
  terminationGracePeriodSeconds >= preStop(10s) + Spring shutdown(25s) + 여유(5s) = 40s 이상
  기본값 30초가 부족할 수 있는 이유: preStop(10s) + Spring shutdown(25s) + 여유(5s) = 40s > 30s → 유예 시간 초과 시 SIGKILL 가능
```

> **근거**: Google Cloud Blog — *"Pod 종료 시 SIGTERM과 Endpoint 제거는 동시에 시작된다. preStop hook으로 앱이 Endpoint 제거보다 먼저 죽는 것을 방지해야 한다."* CNCF Blog — *"terminationGracePeriodSeconds는 전체 종료 과정(preStop + SIGTERM 처리)을 포함하는 유예 시간이다."* ([Google Cloud - Terminating with grace](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-terminating-with-grace), [CNCF - Pod termination lifecycle](https://www.cncf.io/blog/2024/12/19/decoding-the-pod-termination-lifecycle-in-kubernetes-a-comprehensive-guide/))

### 모니터링 — JVM Metrics

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

### HPA와 JVM Warmup 문제

HPA(Horizontal Pod Autoscaler)는 메트릭 기반으로 Pod 수를 자동 조절한다. 그런데 JVM 앱에서는 **Cold Start 문제**가 있다.

```text
트래픽 스파이크
  → HPA가 새 Pod 생성
    → 새 Pod의 JVM은 JIT 미컴파일 상태 → CPU 급등
      → readiness가 너무 빨리 열리면 HPA가 이를 부하로 볼 수 있음
        → 새 Pod도 차가운 상태 → CPU 급등
          → 조건에 따라 과도한 스케일 아웃 가능
```

**원인**: JVM은 JIT(Just-In-Time) 컴파일러를 사용한다. 시작 직후에는 인터프리터 모드로 실행하다가 자주 호출되는 코드를 점진적으로 네이티브 코드로 컴파일한다. 이 워밍업 구간에서 인터프리터 실행과 JIT 컴파일이 동시에 발생하면서 CPU 사용량이 급증한다.

**대응 전략**:
1. **Application-Level Warmup**: 시작 시 주요 코드 경로를 미리 호출하여 JIT 유도. Readiness Probe를 warmup 완료 후에만 성공시킴.
2. **HPA 튜닝**: readiness 지연, CPU initialization period, scale-up 정책을 서비스 warmup 시간에 맞춘다.
3. **CRaC (Coordinated Restore at Checkpoint)**: 워밍업된 JVM 스냅샷을 저장/복원하여 시작 시간을 수초로 단축.

> **근거**: JVM warmup 중 CPU 사용량이 높아질 수 있다는 점은 JVM 애플리케이션 운영에서 알려진 이슈다. Kubernetes HPA에는 `--horizontal-pod-autoscaler-initial-readiness-delay`(기본 30초)와 `--horizontal-pod-autoscaler-cpu-initialization-period`(기본 5분)를 통해 아직 준비되지 않은 Pod의 CPU를 보수적으로 다루는 보호 장치가 있다. 다만 Readiness가 실제 warmup보다 빨리 성공하거나 클러스터 설정이 애플리케이션 warmup 시간과 맞지 않으면 초기 CPU 스파이크가 스케일링 판단에 영향을 줄 수 있다. ([Kubernetes - HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/), [BlaBlaCar - Java and Kubernetes warmup](https://medium.com/blablacar/warm-up-the-relationship-between-java-and-kubernetes-7fc5741f9a23))

---

## 2. CPU와 Memory는 왜 다르게 동작하는가

Kubernetes Pod에는 컨테이너별로 `requests`(기본 점유량)와 `limits`(최대 점유량)를 설정할 수 있다.

```text
Pod Resources 설정 구조:

  resources:
    requests:          ← 스케줄링에 관여 (이만큼 여유 있는 노드에 배치)
      cpu: "500m"
      memory: "1Gi"
    limits:            ← 런타임 상한 (이 이상 사용 시 제재)
      cpu: "2"
      memory: "1Gi"
```

### CPU limits 초과: 느려지지만 죽지 않는다

```text
┌─────────────────────────────────────────┐
│              Node (8 cores)             │
│                                         │
│  Pod A        Pod B        Pod C        │
│  limit: 2     limit: 2     limit: 2     │
│  usage: 3 ⚠   usage: 1     usage: 1     │
│       │                                 │
│       ▼                                 │
│  CPU throttling 발생                     │
│  → cgroup의 CFS 스케줄러가                 │
│    할당 시간을 제한                         │
│  → Pod는 살아있지만 느려짐                   │
│  → 해당 Pod의 응답 지연 가능                 │
│                                         │
│  ✅ Pod 종료: 없음                        │
└─────────────────────────────────────────┘
```

**왜 안 죽는가?** CPU는 **압축 가능한(compressible) 자원**이다. CFS(Completely Fair Scheduler)가 cgroup 단위로 CPU 시간을 시분할(time-slicing) 배분하며, 할당량을 초과하면 다음 주기까지 대기(throttling)시킨다. cgroup v1에서는 주로 `cpu.cfs_quota_us` / `cpu.cfs_period_us`, cgroup v2에서는 `cpu.max`로 이 한도를 표현한다. CPU 시간이 부족할 뿐 프로세스를 종료할 이유가 없다.

> **근거**: Kubernetes 공식 문서 — *"CPU는 compressible resource로, Pod는 CPU 제한 초과 시 throttle된다."* Sysdig 기술 블로그 — *"CPU가 이슈인 경우, 컨테이너는 크래시하지 않고 느리게 응답한다."* ([Kubernetes Docs](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/), [Sysdig Blog](https://www.sysdig.com/blog/troubleshoot-kubernetes-oom))

### Memory limits 초과: OOMKilled될 수 있다

```text
┌─────────────────────────────────────────┐
│              Node (32Gi)                │
│                                         │
│  Pod A        Pod B        Pod C        │
│  limit: 8Gi   limit: 8Gi   limit: 8Gi   │
│  usage: 9Gi ⚠  usage: 6Gi  usage: 6Gi   │
│       │                                 │
│       ▼                                 │
│  cgroup 메모리 한도 초과 감지                │
│  → Linux OOM Killer 발동                 │
│  → SIGKILL (signal 9) 전송               │
│  → 컨테이너 종료 (exit code 137)           │
│                                         │
│  💀 Pod 종료: OOMKilled                  │
└─────────────────────────────────────────┘
```

**왜 죽는가?** Memory는 **비압축(incompressible) 자원**이다. 이미 할당된 메모리를 시간 분할로 나눠 쓸 수 없다. 컨테이너가 memory limit을 초과하고 커널이 메모리 압박을 감지하면 OOM Killer가 프로세스에 SIGKILL을 보내 강제 종료할 수 있다. 이 경우 프로세스에게 정리할 기회(graceful shutdown)가 주어지지 않는다.

> **근거**: Kubernetes 공식 문서 — *"컨테이너가 메모리 limit을 초과하면 종료 대상이 된다."* ([Assign Memory Resources](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/))

### 그래서 설정 전략이 다르다

```text
┌──────────────────────────────────────────────────────────┐
│                   리소스 설정 전략                           │
│                                                          │
│  CPU:  request는 실측 기반, limit은 신중히                │
│        ┌──────────────────────────────────────┐          │
│        │ request: 평상시 필요량                │          │
│        │ limit: 지연 민감도와 노드 정책에 따라  │          │
│        └──────────────────────────────────────┘          │
│        이유: CPU limit은 죽이지는 않지만 throttling으로  │
│              latency를 악화시킬 수 있음                  │
│                                                          │
│  Memory: limit과 JVM 총량을 반드시 맞춘다                 │
│        ┌──────────────────────────────────────┐          │
│        │ heap + metaspace + direct + stack +  │          │
│        │ code cache + native + page cache ... │          │
│        └──────────────────────────────────────┘          │
│        이유: memory limit 초과는 OOMKilled로 이어질 수 있음│
└──────────────────────────────────────────────────────────┘
```

CPU와 Memory는 운영 전략이 다르다.

- **CPU request**는 스케줄링 기준이므로 평상시 필요량과 목표 bin packing을 기준으로 잡는다.
- **CPU limit**은 throttling을 만들 수 있으므로 latency-sensitive 서비스에서는 크게 잡거나, 클러스터 정책이 허용한다면 두지 않는 전략도 검토한다.
- **Memory request**는 노드 배치와 eviction 우선순위에 영향을 준다.
- **Memory limit**은 컨테이너의 실제 상한이므로 JVM의 heap만이 아니라 metaspace, thread stack, direct buffer, code cache, native memory, mmap, page cache, tmpfs `emptyDir`까지 고려해 잡아야 한다.

QoS를 `Guaranteed`로 받고 싶다면 조건이 더 엄격하다. 모든 컨테이너에 CPU와 Memory의 request/limit이 모두 설정되어야 하고, 각 리소스별 request와 limit이 같아야 한다. 즉 Memory만 `requests=limits`로 맞추고 CPU는 `requests < limits`로 두면 `Guaranteed`가 아니라 `Burstable`이다.

```yaml
# Guaranteed QoS 예시
resources:
  requests:
    cpu: "2"
    memory: "2Gi"
  limits:
    cpu: "2"
    memory: "2Gi"

# Burstable QoS 예시
resources:
  requests:
    cpu: "500m"
    memory: "2Gi"
  limits:
    cpu: "2"
    memory: "2Gi"
```

> **근거**: Kubernetes QoS 분류 — `Guaranteed` QoS는 모든 컨테이너의 CPU와 memory request/limit이 모두 존재하고, 각각 request와 limit이 같아야 한다. 이 조건을 만족하지 않고 하나 이상의 request/limit이 있으면 일반적으로 `Burstable`로 분류된다. Node pressure 상황에서는 `BestEffort`, `Burstable`, `Guaranteed` 순서로 축출 우선순위가 높다. ([Kubernetes QoS](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/), [Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/))
