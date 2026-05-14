---
title: "Kubernetes Pod 리소스와 JVM 메모리 — 왜 Pod가 죽고, 왜 힙덤프가 안 남는가"
date: 2026-05-08 10:00:00 +0900
categories: [JVM, Kubernetes]
tags: [kubernetes, jvm, oom, heap-dump, pod-resources, memory, cpu-throttling, alwayspretouch, probe, graceful-shutdown, hpa, monitoring]
---

# Kubernetes Pod 리소스와 JVM 메모리 — 왜 Pod가 죽고, 왜 힙덤프가 안 남는가

> **TL;DR**: K8S에서 JVM 앱을 운영할 때 CPU limit 초과와 Memory limit 초과는 커널의 처리 방식이 다르다. CPU limit 초과는 CFS throttling으로 제한되고, memory limit 초과는 커널 OOM Killer가 SIGKILL을 전송해 프로세스를 강제 종료할 수 있다. JVM OOM과 Pod OOM은 감지 주체, 증상, 대응 방법이 모두 다르다.

<pre>
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
│  CPU는 시분할 가능(compressible), Memory는 시분할 불가(incompressible)  │
└─────────────────────────────────────────────────────────┘
</pre>

<pre>
┌─────────────────────────────────────────────────────────┐
│          Memory 초과 시, 누가 먼저 감지하느냐?          │
├────────────────────────────┬────────────────────────────┤
│                            │                            │
│     JVM 내부 영역 초과     │  JVM 전체 합 > Pod limit   │
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
│   │  힙덤프 생성: O    │   │   │  힙덤프 생성: X    │   │
│   │  로그 기록:  O     │   │   │  로그 기록:  X     │   │
│   │  프로세스: 존속    │   │   │  프로세스: 강제종료│   │
│   │  (좀비 위험)       │   │   │  (JVM 덤프 불가)   │   │
│   └─────────┬──────────┘   │   └─────────┬──────────┘   │
│             │              │             │              │
│             v              │             v              │
│   ┌────────────────────┐   │   ┌────────────────────┐   │
│   │  대응 방법         │   │   │  대응 방법         │   │
│   │                    │   │   │                    │   │
│   │  - 힙덤프 분석     │   │   │  - JVM 옵션 조정   │   │
│   │  - 메모리 누수     │   │   │  - Pod Memory >    │   │
│   │    추적            │   │   │    JVM 총량 보장   │   │
│   │  - ExitOnOOM       │   │   │  - AlwaysPreTouch  │   │
│   │    (좀비 방지)     │   │   │    (RSS 예측)      │   │
│   └────────────────────┘   │   └────────────────────┘   │
│                            │                            │
└────────────────────────────┴────────────────────────────┘
</pre>

> **근거**: Kubernetes 공식 문서는 CPU limit을 커널이 강제하는 hard limit으로 설명하고, CPU 사용량은 throttling으로 제한된다고 설명한다. Memory limit은 OOM kill로 강제되지만, 커널이 메모리 압박을 감지할 때 반응적으로 적용되므로 초과 즉시 항상 종료되는 것은 아니다. JVM OOM과 Pod OOM의 차이는 예외를 던지는 주체(JVM vs 커널)가 다르기 때문에 발생한다. ([Kubernetes Docs - Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/), [HeapHero - OOMKilled vs Java OOM](https://blog.heaphero.io/oomkilled-vs-java-oom-kubernetes/))

---

## 1. K8S에서 JVM 앱은 어떻게 운영되는가

CPU/Memory 동작 차이와 OOM 문제를 이해하려면, K8S가 JVM 앱을 배포·실행·종료하는 전체 라이프사이클을 먼저 파악해야 한다.

### 클러스터 구조

```
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

### JVM 앱의 라이프사이클

```
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

```
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

```
JVM OOM 후 좀비 상태 — Liveness Probe가 없을 때:

  Client ──요청──▶ Pod (좀비)
                    │
                    ├── 프로세스: 살아있음 (PID 존재)
                    ├── Heap: 꽉 참 (OutOfMemoryError 발생)
                    ├── 새 요청마다: 500 Internal Server Error
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
                    ├── ExitOnOutOfMemoryError로 JVM 종료 → 컨테이너 종료
                    │
                    ▼
                kubelet: restartPolicy에 따라 컨테이너 재시작
```

#### Readiness Probe — "트래픽 받을 준비 되었나?"

실패 시 Service의 Endpoint 목록에서 **제거**된다. 재시작이 아니라 **트래픽만 끊는다**. 일시적으로 요청을 처리할 수 없는 상황에서 다른 정상 Pod로 트래픽을 우회시키는 것이 목적이다.

```
DB 장애 시 — Readiness Probe가 없을 때:

  ┌── Service (kube-proxy) ──────────────────────────────────┐
  │                                                           │
  │  트래픽 분배:                                             │
  │    Pod A (정상)  ← 33%                                   │
  │    Pod B (DB 연결 끊김) ← 33% → 관련 요청 실패 가능      │
  │    Pod C (정상)  ← 33%                                   │
  │                                                           │
  │  결과: 일부 요청이 실패할 수 있음                         │
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
  │  결과: 준비된 Pod만 트래픽 수신                           │
  │  DB 복구되면 → Readiness 성공 → 자동으로 Endpoint 복귀   │
  └───────────────────────────────────────────────────────────┘
```

**Liveness vs Readiness를 잘못 쓰면?**

```
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
| **Readiness** | Endpoint에서 **제거** (재시작 아님) | 일시적 장애 시 트래픽 우회 | 필요 시 포함 (DB, 캐시) |

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

```
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

**Race Condition**: Track A(Endpoint/라우팅 반영)보다 Track B(앱 종료)가 먼저 끝나면, 일부 경로에서 종료 중인 Pod로 요청이 갈 수 있다. `preStop: sleep 10` 같은 지연은 이 경합 윈도우를 축소하는 완화책이며, 실제 값은 Ingress/LB/Service 전파 지연과 앱 종료 시간을 기준으로 산정해야 한다.

```
타이밍 공식:
  terminationGracePeriodSeconds >= preStop(10s) + Spring shutdown(25s) + 여유(5s) = 40s 이상
  기본값 30초가 부족할 수 있는 이유: preStop(10s) + Spring shutdown(25s) + 여유(5s) = 40s > 30s → 유예 시간 초과 시 SIGKILL 가능
```

> **근거**: Google Cloud Blog — *"Pod 종료 시 SIGTERM과 Endpoint 제거는 동시에 시작된다. preStop hook으로 앱이 Endpoint 제거보다 먼저 죽는 것을 방지해야 한다."* CNCF Blog — *"terminationGracePeriodSeconds는 전체 종료 과정(preStop + SIGTERM 처리)을 포함하는 유예 시간이다."* ([Google Cloud - Terminating with grace](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-terminating-with-grace), [CNCF - Pod termination lifecycle](https://www.cncf.io/blog/2024/12/19/decoding-the-pod-termination-lifecycle-in-kubernetes-a-comprehensive-guide/))

### 모니터링 — JVM Metrics

```
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

```
트래픽 스파이크
  → HPA가 새 Pod 생성
    → 새 Pod의 JVM은 JIT 미컴파일 상태 → CPU 급등
      → HPA가 이를 "부하 증가"로 오인 → 추가 Pod 생성
        → 새 Pod도 차가운 상태 → CPU 급등
          → 과도한 스케일 아웃 가능
```

**원인**: JVM은 JIT(Just-In-Time) 컴파일러를 사용한다. 시작 직후에는 인터프리터 모드로 실행하다가 자주 호출되는 코드를 점진적으로 네이티브 코드로 컴파일한다. 이 워밍업 구간에서 인터프리터 실행과 JIT 컴파일이 동시에 발생하면서 CPU 사용량이 급증한다.

**대응 전략**:
1. **Application-Level Warmup**: 시작 시 주요 코드 경로를 미리 호출하여 JIT 유도. Readiness Probe를 warmup 완료 후에만 성공시킴.
2. **HPA 튜닝**: readiness 지연, CPU initialization period, scale-up 정책을 서비스 warmup 시간에 맞춘다.
3. **CRaC (Coordinated Restore at Checkpoint)**: 워밍업된 JVM 스냅샷을 저장/복원하여 시작 시간을 수초로 단축.

> **근거**: JVM warmup 중 CPU 사용량이 높아질 수 있다는 점은 JVM 애플리케이션 운영에서 알려진 이슈다. Kubernetes HPA에는 readiness 지연과 CPU initialization period를 통해 아직 준비되지 않은 Pod의 CPU를 보수적으로 다루는 보호 장치가 있다. ([Kubernetes - HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/), [BlaBlaCar - Java and Kubernetes warmup](https://medium.com/blablacar/warm-up-the-relationship-between-java-and-kubernetes-7fc5741f9a23))

---

## 2. CPU와 Memory는 왜 다르게 동작하는가

Kubernetes Pod에는 컨테이너별로 `requests`(기본 점유량)와 `limits`(최대 점유량)를 설정할 수 있다.

```
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

```
┌─────────────────────────────────────────┐
│              Node (8 cores)             │
│                                         │
│  Pod A        Pod B        Pod C        │
│  limit: 2     limit: 2     limit: 2    │
│  usage: 3 ⚠   usage: 1     usage: 1   │
│       │                                 │
│       ▼                                 │
│  CPU throttling 발생                    │
│  → cgroup의 CFS 스케줄러가             │
│    할당 시간을 제한                      │
│  → Pod는 살아있지만 느려짐              │
│  → 해당 Pod의 응답 지연 가능             │
│                                         │
│  ✅ Pod 종료: 없음                      │
└─────────────────────────────────────────┘
```

**왜 안 죽는가?** CPU는 **압축 가능한(compressible) 자원**이다. CFS(Completely Fair Scheduler)가 cgroup 단위로 CPU 시간을 시분할(time-slicing) 배분하며, 할당량(`cpu.cfs_quota_us`)을 초과하면 다음 주기(`cpu.cfs_period_us`)까지 대기(throttling)시킨다. CPU 시간이 부족할 뿐 프로세스를 종료할 이유가 없다.

> **근거**: Kubernetes 공식 문서 — *"CPU는 compressible resource로, Pod는 CPU 제한 초과 시 throttle된다."* Sysdig 기술 블로그 — *"CPU가 이슈인 경우, 컨테이너는 크래시하지 않고 느리게 응답한다."* ([Kubernetes Docs](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/), [Sysdig Blog](https://www.sysdig.com/blog/troubleshoot-kubernetes-oom))

### Memory limits 초과: OOMKilled될 수 있다

```
┌─────────────────────────────────────────┐
│              Node (32Gi)                │
│                                         │
│  Pod A        Pod B        Pod C        │
│  limit: 8Gi   limit: 8Gi   limit: 8Gi │
│  usage: 9Gi ⚠  usage: 6Gi  usage: 6Gi │
│       │                                 │
│       ▼                                 │
│  cgroup 메모리 한도 초과 감지            │
│  → Linux OOM Killer 발동               │
│  → SIGKILL (signal 9) 전송             │
│  → 컨테이너 종료 (exit code 137)       │
│                                         │
│  💀 Pod 종료: OOMKilled                 │
└─────────────────────────────────────────┘
```

**왜 죽는가?** Memory는 **비압축(incompressible) 자원**이다. 이미 할당된 메모리를 시간 분할로 나눠 쓸 수 없다. 컨테이너가 memory limit을 초과하고 커널이 메모리 압박을 감지하면 OOM Killer가 프로세스에 SIGKILL을 보내 강제 종료할 수 있다. 이 경우 프로세스에게 정리할 기회(graceful shutdown)가 주어지지 않는다.

> **근거**: Kubernetes 공식 문서 — *"컨테이너가 메모리 limit을 초과하면 종료 대상이 된다."* ([Assign Memory Resources](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/))

### 그래서 설정 전략이 다르다

```
┌──────────────────────────────────────────────────────────┐
│                   리소스 설정 전략                         │
│                                                          │
│  CPU:  requests ≪ limits                                │
│        ┌──────────────────────────────────────┐          │
│        │ requests: 0.5    limits: 4           │          │
│        │ ├─────┤          ├──────────────────┤ │          │
│        │ 평상시 사용량     배포/스파이크 대비   │          │
│        └──────────────────────────────────────┘          │
│        이유: 배포 시점에 CPU를 많이 사용하므로             │
│              limits는 충분히, requests는 적게.             │
│              requests=limits로 높게 잡으면 → 리소스 낭비  │
│              requests=limits로 낮게 잡으면 → 배포 지연    │
│                                                          │
│  Memory: requests = limits (동일하게)                    │
│        ┌──────────────────────────────────────┐          │
│        │ requests = limits = 2Gi              │          │
│        │ ├──────────────────┤                 │          │
│        │ JVM 내부 여유를 계산해 고정             │          │
│        └──────────────────────────────────────┘          │
│        이유: requests < limits로 잡으면                   │
│              실사용량이 requests를 넘을 때                 │
│              노드 OOM → 연쇄 축출 위험                    │
└──────────────────────────────────────────────────────────┘
```

> **근거**: Kubernetes QoS 분류 — CPU와 memory 모두 requests=limits로 설정하면 `Guaranteed` QoS를 받아 노드 압박 상황에서 축출 우선순위가 가장 낮다. requests < limits이면 `Burstable`로 분류되고, 사용량이 requests를 초과한 Pod는 노드 리소스 부족 시 축출 후보가 될 수 있다. ([Kubernetes QoS](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/), [Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/))

---

## 3. 노드 OOM의 연쇄 축출 — 왜 하나가 죽으면 전체가 위험한가

Memory를 `requests < limits`로 설정하면 스케줄러는 requests 기준으로 노드를 배치한다. 실 사용량이 requests를 초과해 limits까지 도달하면, 스케줄러가 산정한 가용 공간과 실제 물리 메모리 가용량 사이에 괴리가 발생한다.

```
초기 상태 — 스케줄러는 requests만 본다
┌───────────────── Node 1 (32Gi) ─────────────────┐
│                                                   │
│  Pod A          Pod B          Pod C              │
│  req: 4Gi       req: 4Gi       req: 4Gi          │
│  lim: 10Gi      lim: 10Gi      lim: 10Gi         │
│  실사용: 8Gi    실사용: 9Gi    실사용: 9Gi        │
│                                                   │
│  스케줄러가 본 점유: 12Gi / 32Gi (여유 있음)       │
│  실제 사용량:        26Gi / 32Gi (거의 꽉 참)      │
└───────────────────────────────────────────────────┘

         Pod B가 10Gi까지 사용 → 노드 총 사용량 27Gi
         → 노드 memory.available 감소
         → kubelet eviction threshold 도달
                    │
                    ▼
┌───────────────── Node 1 ──────────────────────────┐
│                                                    │
│  Pod A ✅        Pod B 💀축출     Pod C ✅         │
│                  → Node 2로 재스케줄링              │
└────────────────────────────────────────────────────┘
                    │
                    ▼
┌───────────────── Node 2 (32Gi) ─────────────────┐
│                                                   │
│  기존 Pod들 + 축출된 Pod B                        │
│  → Node 2도 실 사용량 급증                        │
│  → Node 2에서도 eviction 발생 가능                │
│  → 연쇄 축출 (Cascade Eviction)                   │
└───────────────────────────────────────────────────┘
```

> **근거**: Kubernetes 공식 문서 — kubelet은 `memory.available`이 eviction threshold 이하로 떨어지면 QoS와 사용량 기준으로 Pod를 축출한다. 축출된 Pod가 컨트롤러에 의해 다른 노드에 재생성되면 해당 노드의 부하가 증가할 수 있다. 연쇄 축출 설명은 이 동작에서 파생되는 운영상 시나리오다. ([Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/), [Pod QoS](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/))

**결론**: JVM 서비스처럼 메모리 사용량 예측과 축출 안정성이 중요한 워크로드는 memory를 **requests = limits**로 설정해 `Guaranteed` QoS를 확보하는 전략이 유리하다. 다만 이 값 안에는 Heap뿐 아니라 Metaspace, CodeCache, thread stack, direct buffer, native memory 여유까지 포함해야 한다.

---

## 4. JVM 메모리 구조 — Pod 안에서 무슨 일이 일어나는가

JVM은 Pod에 할당된 메모리 내에서 Heap, Metaspace, CodeCache, Thread Stack 등 독립적인 메모리 영역을 관리한다.

```
┌──────────────────── Pod Memory (requests=limits) ─────────────────────┐
│                                                                       │
│  ┌─────────────────── JVM Process Memory ──────────────────────────┐  │
│  │                                                                  │  │
│  │  ┌──────────────────────────────────────────┐                   │  │
│  │  │              Heap                         │                   │  │
│  │  │  new()로 생성된 모든 객체 인스턴스          │                   │  │
│  │  │                                           │                   │  │
│  │  │  설정: -XX:MaxRamPercentage               │                   │  │
│  │  │  실제크기 = Pod Memory * Percentage / 100  │                   │  │
│  │  └──────────────────────────────────────────┘                   │  │
│  │                                                                  │  │
│  │  ┌──────────────┐  ┌───────────┐  ┌──────────────────────────┐  │  │
│  │  │  Metaspace   │  │ CodeCache │  │     Thread Stacks        │  │  │
│  │  │              │  │           │  │                          │  │  │
│  │  │ 클래스 메타   │  │ JIT 컴파일│  │ Thread 수 * 1MB (기본)   │  │  │
│  │  │ 데이터 저장   │  │ 네이티브  │  │ -Xss로 설정              │  │  │
│  │  │              │  │ 코드 저장 │  │                          │  │  │
│  │  │ 설정:        │  │           │  │ 가변 (Thread 수에 비례)   │  │  │
│  │  │ MaxMetaspace │  │ Reserved  │  │                          │  │  │
│  │  │ Size         │  │ CodeCache │  │                          │  │  │
│  │  │              │  │ Size      │  │                          │  │  │
│  │  └──────────────┘  └───────────┘  └──────────────────────────┘  │  │
│  │                                                                  │  │
│  │  + GC, Compiler, Internal 등 (소량)                              │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  + 컨테이너 OS 오버헤드 (α)                                          │
└───────────────────────────────────────────────────────────────────────┘
```

### 각 영역 상세

| 영역 | 저장 내용 | 설정 옵션 | 기본값 | 특성 |
|------|----------|----------|--------|------|
| **Heap** | `new()`로 생성된 객체 인스턴스 | `-XX:MaxRAMPercentage` | 서버 VM 기본 25% 계열 | GC 대상, 가장 큰 영역 |
| **Metaspace** | ClassLoader가 로드한 클래스 메타데이터 | `-XX:MaxMetaspaceSize` | 명시 상한 없음 | 클래스 수에 비례 |
| **CodeCache** | JIT 컴파일된 네이티브 코드 | `-XX:ReservedCodeCacheSize` | JDK/VM 설정에 따라 다름 | Tiered Compilation 사용 시 세그먼트 분리 |
| **Thread Stack** | Thread별 호출 스택 | `-Xss` | 플랫폼/JDK에 따라 다름 | Thread 수에 비례 (가변) |

#### Metaspace 보충 설명

Metaspace는 내부적으로 두 공간으로 나뉜다:

```
┌─────────── MaxMetaspaceSize ──────────────┐
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │     Compressed Class Space           │  │
│  │     (Klass 포인터만 저장)             │  │
│  │     default: 1GB                     │  │
│  │     MaxMetaspaceSize 설정 시          │  │
│  │     자동 조정됨                       │  │
│  └──────────────────────────────────────┘  │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │     Non-Class Metadata               │  │
│  │     (Method, Constant Pool,          │  │
│  │      Annotations 등)                 │  │
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

`MaxMetaspaceSize`를 설정하면 Compressed Class Space + Non-Class Metadata 합산에 영향을 주는 상한이 된다. `CompressedClassSpaceSize`를 별도로 설정하지 않으면 JVM이 내부 규칙에 따라 조정하므로, 특정 비율로 고정된다고 보면 안 된다.

> **근거**: Oracle GC Tuning Guide — *"MaxMetaspaceSize는 committed compressed class space와 기타 class metadata 공간의 합에 적용된다."* Red Hat Developer — *"MaxMetaspaceSize가 CompressedClassSpaceSize보다 작으면 JVM이 자동 조정한다."* ([Oracle Docs](https://docs.oracle.com/en/java/javase/11/gctuning/other-considerations.html), [Red Hat Developer](https://developers.redhat.com/articles/2024/07/19/metaspace-setting-and-tuning-jdk-8-applications-and-outside-containers), [stuefe.de - Sizing Metaspace](https://stuefe.de/posts/metaspace/sizing-metaspace/))

#### CodeCache 보충 설명

Tiered Compilation이 활성화되고 `ReservedCodeCacheSize >= 240MB`이면 CodeCache가 3개 세그먼트로 분리된다:

| 세그먼트 | 내용 | 수명 |
|---------|------|------|
| non-method | 컴파일러 버퍼, 바이트코드 인터프리터 | 영구 (고정 ~3MB) |
| profiled nmethod | 가볍게 최적화된 프로파일링 메서드 | 짧음 |
| non-profiled nmethod | 완전 최적화된 메서드 | 길음 |

> **근거**: Baeldung — *"Tiered compilation이 활성화되고 ReservedCodeCacheSize >= 240MB이면 세그먼트화가 기본 적용된다."* Oracle JDK 8 Embedded Docs — *"non-method 영역은 고정 크기 ~3MB다."* ([Baeldung - JVM Code Cache](https://www.baeldung.com/jvm-code-cache), [Jason Pearson - CodeCache](https://www.jasonpearson.dev/codecache-in-jvm-builds/))

---

## 5. JVM OOM vs Pod OOM — 같은 OOM, 다른 결과

이 구분이 운영에서 가장 중요하다. 같은 "메모리 부족"이지만 원인, 증상, 대응이 완전히 다르다.

### Case 1: JVM OOM (JVM 레벨)

```
┌──────────── Pod Memory: 2Gi ─────────────┐
│                                           │
│  ┌──────── JVM Memory ────────────────┐   │
│  │                                     │   │
│  │  Heap: max 1Gi                     │   │
│  │  ┌─────────────────────────────┐   │   │
│  │  │ ██████████████████████ 1Gi  │ ⚠ │   │  ← Heap 꽉 참
│  │  └─────────────────────────────┘   │   │
│  │                                     │   │
│  │  Metaspace: 128MB    ✅ 여유 있음   │   │
│  │  CodeCache: 100MB    ✅ 여유 있음   │   │
│  │  Stacks: 200MB       ✅ 여유 있음   │   │
│  │                                     │   │
│  │  JVM 총 사용: ~1.4Gi < Pod 2Gi ✅  │   │
│  └─────────────────────────────────────┘   │
│                                           │
│  Pod 메모리 여유: 있음                     │
│  BUT Heap 영역이 개별적으로 초과           │
│                                           │
│  → java.lang.OutOfMemoryError 발생        │
│  → JVM이 스스로 예외를 던짐                │
│  → HeapDumpOnOutOfMemoryError 동작 ✅     │
│  → 힙덤프 분석 가능 ✅                    │
└───────────────────────────────────────────┘
```

**특징**:
- JVM **자체**가 메모리 할당 실패를 감지하고 `OutOfMemoryError`를 던진다
- 프로세스는 **계속 살아있을 수 있다** (일반 예외처럼 처리되면 프로세스 종료가 보장되지 않음)
- `-XX:+HeapDumpOnOutOfMemoryError` → JVM이 OOME를 처리할 시간이 있으면 힙덤프 생성 **가능**
- `-XX:+ExitOnOutOfMemoryError` 없으면 장애 상태로 남을 수 있음

**장애 지속 문제**: JVM OOM이 발생해도 프로세스 종료가 보장되지는 않는다. Spring Boot Actuator의 health check 엔드포인트(`/actuator/health`)가 정상 응답을 반환하는 경우도 있다. 이때 liveness probe가 계속 통과하면 Kubernetes는 Pod를 재시작하지 않으므로, 요청은 받지만 일부 또는 대부분의 비즈니스 처리가 실패하는 상태가 지속될 수 있다.

```
JVM OOM 후 좀비 상태:

  Client ──요청──▶ Pod (좀비)
                    │
                    ├── liveness probe: /actuator/health → 200 OK일 수 있음
                    ├── 실제 요청 처리: OutOfMemoryError → 500 ❌
                    │
                    └── Kubernetes: "Pod 정상이니 재시작 안 함"

  해결: -XX:+ExitOnOutOfMemoryError
        → OOM 발생 시 JVM 종료
        → exit code != 0
        → restartPolicy에 따라 컨테이너 재시작
```

> **근거**: HeapHero 블로그 — *"JVM OutOfMemoryError는 JVM이 스스로 던지는 예외이며, 프로세스는 여전히 살아있다. HeapDumpOnOutOfMemoryError는 JVM이 OutOfMemoryError를 던질 때만 동작한다."* ([HeapHero - OOMKilled vs Java OOM](https://blog.heaphero.io/oomkilled-vs-java-oom-kubernetes/))

### Case 2: Pod OOM (Kubernetes 레벨)

```
┌──────────── Pod Memory: 2Gi ─────────────┐
│                                           │
│  ┌──────── JVM Memory ────────────────┐   │
│  │                                     │   │
│  │  Heap: max 1.2Gi   사용: 1.0Gi ✅  │   │
│  │  Metaspace: 128MB   사용: 120MB ✅  │   │
│  │  CodeCache: 240MB   사용: 100MB ✅  │   │
│  │  Stacks: 500MB      (500 threads)  │   │
│  │  기타: 200MB                       │   │
│  │                                     │   │
│  │  JVM 총 사용: ~2.1Gi               │   │
│  └──────────────────────┼──────────────┘   │
│                         │                  │
│  Pod Memory limit: 2Gi  │ 초과! ⚠         │
│                         ▼                  │
│  Linux cgroup 메모리 한도 초과 감지         │
│  → OOM Killer 발동                        │
│  → SIGKILL (signal 9) → 강제 종료         │
│  → exit code 137                          │
│                                           │
│  HeapDumpOnOutOfMemoryError? ❌ 동작 안 함│
│  → JVM이 OOM을 감지한 게 아님             │
│  → 커널이 프로세스를 죽인 것               │
│  → JVM 힙덤프 생성 불가 ❌                │
└───────────────────────────────────────────┘
```

**왜 힙덤프가 안 남는가?**

`HeapDumpOnOutOfMemoryError`는 JVM이 `OutOfMemoryError`를 **던지고 처리할 수 있는 시점**에 트리거된다. Pod OOM은 JVM이 아니라 **Linux 커널의 OOM Killer**가 SIGKILL을 보내는 것이다. SIGKILL은 catch 불가능하며, JVM에게 정리할 기회가 없다. 그래서 JVM 힙덤프나 OOME 스택 로그가 남지 않을 수 있고, 원인은 Kubernetes event, container last state, node/kernel 로그에서 확인해야 한다.

> **근거**: HeapHero 블로그 — *"exit code 137은 Linux 커널이 SIGKILL(9)을 보낸 것이며, 깔끔한 종료도 없고 힙덤프도 남지 않는다."* Komodor — *"OOMKilled(137)는 컨테이너가 메모리 limit을 초과했을 때 발생하며, JVM OOM 핸들링 옵션과는 무관하다."* ([HeapHero](https://blog.heaphero.io/oomkilled-vs-java-oom-kubernetes/), [Komodor](https://komodor.com/learn/how-to-fix-oomkilled-exit-code-137/))

### 비교 요약

| 구분 | JVM OOM | Pod OOM (OOMKilled) |
|------|---------|---------------------|
| **누가 감지?** | JVM 자체 | Linux 커널 OOM Killer |
| **시그널** | 예외 (OutOfMemoryError) | SIGKILL (signal 9) |
| **exit code** | ExitOnOOM 시 비정상 종료 코드 | 보통 **137** |
| **힙덤프** | 생성 가능 | JVM 힙덤프 생성 불가 |
| **로그** | JVM 로그에 기록 가능 | JVM OOME 로그는 없음. Kubernetes event/last state 확인 |
| **원인** | Heap/Metaspace 등 개별 JVM 영역 초과 | JVM 프로세스 RSS/컨테이너 메모리가 Pod limit 초과 |
| **대응** | 힙덤프 분석 → 메모리 누수 추적 | JVM 옵션 조정 → Pod 메모리 > JVM 총량 보장 |

---

## 6. RSS를 예측 가능하게 만들기 — AlwaysPreTouch

### RSS란?

RSS(Resident Set Size)는 프로세스가 실제로 물리 메모리에 적재한 크기다. 컨테이너 메모리 제한은 RSS뿐 아니라 cgroup이 집계하는 메모리 사용량 기준으로 적용되므로, 운영에서는 RSS와 cgroup memory usage를 함께 봐야 한다.

```
RSS ≈ Heap committed/used + Metaspace + Thread Stacks + CodeCache + Direct Buffer + mmap/native + α
```

문제는 Heap committed와 native 영역 사용량이 **가변적**이라는 것이다. JVM은 기본적으로 필요할 때 OS에 물리 메모리를 요청한다. 그래서 RSS가 부하에 따라 증가하고, Pod OOM이 "갑자기" 발생하는 것처럼 보일 수 있다.

### AlwaysPreTouch의 역할

```
-XX:+AlwaysPreTouch 없을 때 (기본):

  시간 →  t=0        t=1min      t=5min      t=10min (부하)
  RSS      │          │           │            │
  ─────────┼──────────┼───────────┼────────────┼──────
           │200MB     │400MB      │600MB       │1.8Gi ← 갑자기 급증
           │          │           │            │        Pod OOM 위험!
           │          │           │            │
           └──────────┴───────────┴────────────┘
           Heap이 필요할 때마다 OS에 메모리 요청 → RSS 점진 증가


-XX:+AlwaysPreTouch 있을 때:

  시간 →  t=0        t=1min      t=5min      t=10min (부하)
  RSS      │          │           │            │
  ─────────┼──────────┼───────────┼────────────┼──────
           │1.5Gi     │1.5Gi      │1.55Gi      │1.6Gi ← 안정적
           │          │           │            │
           └──────────┴───────────┴────────────┘
           시작 시 Heap 전체를 미리 RSS에 채움
           → Heap 확장에 따른 RSS 급증 완화
           → 예측 가능성 증가
```

`AlwaysPreTouch`는 JVM 시작 시 초기 heap 영역의 페이지를 미리 터치해 물리 메모리에 반영한다. `InitialRAMPercentage`와 `MaxRAMPercentage`를 같게 두면 시작 시점에 최대 heap에 가까운 RSS가 드러나므로, heap 확장 때문에 런타임에 RSS가 크게 튀는 문제를 줄일 수 있다. 다만 direct buffer, thread stack, code cache, GC/native 영역은 별도로 증가할 수 있으므로 RSS가 완전히 고정되는 것은 아니다.

> **근거**: `AlwaysPreTouch`는 JVM 시작 시 메모리 페이지를 미리 터치하는 옵션이다. 이 옵션은 heap 확장에 따른 RSS 증가를 앞당겨 보이게 하지만, JVM의 모든 native/off-heap 메모리 변동을 없애지는 않는다. ([Oracle java launcher options](https://docs.oracle.com/en/java/javase/17/docs/specs/man/java.html), [JVM Field Guide](https://serce.me/posts/2023-02-01-jvm-field-guide-memory))

**주의**: `InitialRAMPercentage`를 `MaxRAMPercentage`와 동일하게 설정해야 시작 시점의 heap 크기와 최대 heap 크기가 맞는다. 둘이 다르면 AlwaysPreTouch가 시작 시점의 heap에는 적용되더라도, 최대 heap까지 확장되는 구간은 런타임에 추가로 commit/touch될 수 있다.

---

## 7. MinRamPercentage의 이름 함정

`MinRAMPercentage`는 이름과 달리 **"최소 Heap 크기"를 지정하는 옵션이 아니다.**

```
이름이 시사하는 것 (잘못된 이해):
  "Heap의 최소 크기를 전체 RAM의 N%로 설정"

실제 동작:
  "전체 가용 메모리가 ~200MB 이하인 소형 환경에서
   MaxRAMPercentage 대신 사용되는 Heap 상한 비율"

┌───────────────────────────────────────────────────────┐
│                                                       │
│   가용 메모리 > ~200MB  →  MaxRAMPercentage 적용     │
│   가용 메모리 ≤ ~200MB  →  MinRAMPercentage 적용     │
│                                                       │
│   대부분의 서버/컨테이너는 200MB 이상                  │
│   → MinRAMPercentage는 사실상 무시됨                  │
│   → MaxRAMPercentage만 설정하면 충분                  │
│                                                       │
└───────────────────────────────────────────────────────┘
```

위키 원문에서 `MinRamPercentage`와 `MaxRamPercentage`를 동일하게 설정하라고 안내하는데, 이는 **Heap 크기를 고정하려는 의도**로는 맞다. 다만 기술적으로는 컨테이너 메모리가 200MB 이상이면 `MinRAMPercentage`는 적용되지 않으므로, `MaxRAMPercentage`만으로도 Heap 상한을 결정할 수 있다. `InitialRAMPercentage`를 `MaxRAMPercentage`와 동일하게 설정하면 시작 시 Heap 크기도 고정된다.

> **근거**: Baeldung — *"MinRAMPercentage는 소량 메모리(약 200MB 미만) 환경에서 JVM Heap 상한을 결정한다. 대부분의 엔터프라이즈 애플리케이션은 200MB 이상이므로 MinRAMPercentage 설정이 불필요하다."* DZone — *"MinRAMPercentage는 전체 가용 메모리가 약 250MB 미만일 때만 Heap 크기 계산에 사용된다."* ([Baeldung](https://www.baeldung.com/java-jvm-parameters-rampercentage), [DZone](https://dzone.com/articles/difference-between-initialrampercentage-minramperc))

---

## 8. 실제 운영 설정 — apps-manifest 코드 기반

위키 원문의 권장 설정과 실제 운영(real) 환경의 apps-manifest 설정을 비교한다.

### Helm Chart JVM 옵션 구조

현재 shopby-api 차트는 **3.0.x** 버전을 사용하며, `jvmOptions` 블록으로 JVM 메모리를 설정한다.

```yaml
# apps-manifest/chart/real/backend/store/display-shop.yaml (shopby-api 3.0.2)
app:
  heapdump:
    enabled: true              # ExitOnOutOfMemoryError + HeapDump 자동 적용
  prometheusScrape:
    enabled: true              # micrometer-registry-prometheus 메트릭 수집
  jvmOptions:
    ramPercentage: "41"        # MaxRAMPercentage = 41%
    metaspaceSize: "289m"      # MetaspaceSize = MaxMetaspaceSize = 289m
    codeCacheSize: "187m"      # ReservedCodeCacheSize = 187m
resources:
  requests:
    cpu: "0.5"                 # CPU: requests 적게
    memory: "4Gi"              # Memory: requests = limits
  limits:
    cpu: "6"                   # CPU: limits 충분하게
    memory: "4Gi"              # Memory: requests = limits
```

### 실제 운영 서비스별 설정 비교

| 서비스 | Pod Memory | ramPercentage | 실제 Heap | Metaspace | CodeCache | Heap 외 여유 |
|--------|-----------|---------------|----------|-----------|-----------|-------------|
| **display-shop** | 4Gi | 41% | 1,638MB | 289MB | 187MB | **1,966MB (48%)** |
| **product-admin** | 5Gi | 60% | 3,072MB | 331MB | 221MB | **1,496MB (29%)** |
| **product-internal** | 3.2Gi | 42.6% | 1,395MB | 259MB | 155MB | **1,467MB (45%)** |

```
display-shop (real) 메모리 레이아웃:

┌────────────────── Pod Memory: 4Gi ──────────────────────┐
│                                                          │
│  ┌──────── JVM Memory ───────────────────────────────┐   │
│  │                                                    │   │
│  │  Heap: 4Gi * 41% = 1,638MB                       │   │
│  │  ├──────────────────────────┤                      │   │
│  │                                                    │   │
│  │  Metaspace: 289MB   CodeCache: 187MB              │   │
│  │  ├────────────┤     ├──────────┤                   │   │
│  │                                                    │   │
│  │  Stacks + 기타: 가변                               │   │
│  │                                                    │   │
│  │  JVM 고정 합계: ~2,114MB                           │   │
│  └────────────────────────────────────────────────────┘   │
│                                                          │
│  Pod 여유: 4,096 - 2,114 = ~1,982MB (48%) ✅            │
│  → Stacks, GC, Native 등에 충분한 여유                    │
└──────────────────────────────────────────────────────────┘
```

### 위키 권장값 vs 실제 운영값 차이

| 항목 | 위키 권장 | 실제 운영 (display-shop) | 비고 |
|------|----------|------------------------|------|
| ramPercentage | 50% | **41%** | 위키보다 보수적. Heap 외 여유분 더 확보 |
| MetaspaceSize | 128m | **289m** | 위키 예시의 2.3배. 실 사용량 모니터링 기반 |
| CodeCacheSize | 100m (또는 기본 240m) | **187m** | 기본 240m에서 축소, 위키 예시보다는 큼 |
| AlwaysPreTouch | 권장 | **real 미적용** (beta에서만 테스트 중) | 아래 상세 |
| Helm chart | shopby-api:2.0.2 | **shopby-api:3.0.2~3.0.5** | 위키 작성 이후 버전 업그레이드됨 |

**핵심 차이점**: 위키 원문의 예시값(ramPercentage=50%, MetaspaceSize=128m, CodeCacheSize=100m)은 개념 설명용이다. 실제 운영 설정은 Prometheus 대시보드의 JVM Memory Usage 그래프를 모니터링한 후 **실 사용량 기반**으로 튜닝된 값이므로 서비스마다 다르다.

### AlwaysPreTouch: beta에서만 테스트 중

```yaml
# apps-manifest/chart/beta/backend/store/display-shop.yaml
app:
  jvmOptions:
    ramPercentage: '59'
  additionalJvmOptions:
    - -XX:+AlwaysPreTouch
    - -XX:InitialRAMPercentage=50.0    # ⚠ ramPercentage(59%)와 불일치
```

AlwaysPreTouch는 현재 **beta display-shop에서만** 적용 중이며, real 환경에는 미적용 상태다.

또한 beta 설정에서 `ramPercentage: 59%`(MaxRAMPercentage)인데 `InitialRAMPercentage=50.0%`으로 **불일치**한다. 이 경우 AlwaysPreTouch는 50%만큼만 PreTouch하고, 나머지 9%(59%-50%)는 런타임에 필요 시 추가 매핑된다. RSS 예측 가능성 효과가 감소한다.

```
beta display-shop의 실제 동작:

  InitialRAMPercentage=50%
  MaxRAMPercentage=59%

  시작 시 RSS: Pod Memory * 50% = PreTouch 완료
  최대 RSS:    Pod Memory * 59% = Heap 최대
                          ↑
                    이 9% 구간은 런타임에 증가 가능
                    → AlwaysPreTouch 효과 부분 감소

  권장: InitialRAMPercentage = MaxRAMPercentage (= ramPercentage)
```

### heapdump.enabled: OOM 대응 자동화

```yaml
# 전 서비스 공통 (441개 파일)
app:
  heapdump:
    enabled: true
```

`heapdump.enabled: true` 설정 시, shopby-api Helm chart 템플릿이 다음을 자동 주입한다:
- `JAVA_TOOL_OPTIONS`에 `-XX:+ExitOnOutOfMemoryError` 추가
- postStart Hook으로 힙덤프 OBS 업로드 + Dooray 메신저 알림 스크립트 실행

즉, `ExitOnOutOfMemoryError`와 `HeapDumpOnOutOfMemoryError`를 직접 `additionalJvmOptions`에 명시하지 않아도 `heapdump.enabled: true`만으로 적용된다.

### CPU/Memory 설정 패턴 코드 검증

```
CPU: requests ≪ limits (코드 확인 ✅)
───────────────────────────────────────────────────
  display-shop:      cpu requests=0.5  limits=6
  product-admin:     cpu requests=0.5  limits=4
  product-internal:  cpu requests=1    limits=4

Memory: requests = limits (코드 확인 ✅)
───────────────────────────────────────────────────
  display-shop:      memory 4Gi / 4Gi
  product-admin:     memory 5Gi / 5Gi
  product-internal:  memory 3.2Gi / 3.2Gi
```

### Prometheus 메트릭 수집 코드 검증

모든 서비스 모듈에서 `micrometer-registry-prometheus` + `spring-boot-starter-actuator` 의존성 확인:

```
product: shop, server, admin, internal, consumer, batch (6개 모듈)
display: shop, server, admin, internal, batch, sitemap (6개 모듈)
```

Actuator 설정 (`bootstrap.yml`):
```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
    enabled-by-default: true
```

> **근거**: DZone Best Practices — *"어떤 옵션을 사용하든 컨테이너 메모리(-m)의 최소 25%는 Heap 외 영역을 위해 확보해야 한다."* 실제 운영 설정은 이 기준을 충족한다 (display-shop 48%, product-admin 29%, product-internal 45%). ([DZone - Java Memory for Containers](https://dzone.com/articles/best-practices-java-memory-arguments-for-container))

---

## 9. 인과관계 검증 요약

아래는 원문 위키의 주요 인과관계 주장에 대한 사실 확인 결과다.

| # | 원문 주장 | 검증 결과 | 근거 |
|---|----------|----------|------|
| 1 | CPU limit 초과 시 Pod가 안 죽고 느려진다 | **사실** | CPU는 compressible resource, CFS throttling 적용 ([K8S Docs](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)) |
| 2 | Memory limit 초과 시 OOMKilled 가능 | **사실** | 커널 OOM Killer가 SIGKILL 전송, 보통 exit code 137 ([K8S Docs](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)) |
| 3 | 축출된 Pod가 다른 노드로 전이되어 연쇄 OOM 가능 | **가능한 운영 시나리오** | kubelet eviction + 컨트롤러 재생성으로 다른 노드 부하가 증가할 수 있음 ([K8S Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)) |
| 4 | Memory requests=limits가 JVM 운영에 유리하다 | **조건부 권장** | Guaranteed QoS 확보. 단, Heap 외 메모리 여유까지 포함해 산정해야 함 ([K8S QoS](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)) |
| 5 | CPU는 requests보다 limits를 크게 둘 수 있다 | **상황별 권장** | CPU burst를 허용할 수 있지만 throttling과 비용 정책을 함께 봐야 함 |
| 6 | MaxRAMPercentage로 Heap 상한 비율을 조정한다 | **사실** | 컨테이너 메모리 인식 환경에서 최대 heap 비율 조정 가능 ([Oracle java options](https://docs.oracle.com/en/java/javase/17/docs/specs/man/java.html)) |
| 7 | Metaspace 기본값에 명시 상한이 없다 | **사실** | `MaxMetaspaceSize` 미설정 시 별도 상한 없이 native memory 영역에서 확장 가능 ([Oracle Docs](https://docs.oracle.com/en/java/javase/11/gctuning/other-considerations.html)) |
| 8 | Compressed Class Space가 MaxMetaspaceSize의 약 81.2% | **부정확** | 고정 비율이 아니며 JVM 내부 규칙과 설정에 따라 달라짐 ([stuefe.de](https://stuefe.de/posts/metaspace/sizing-metaspace/)) |
| 9 | CodeCache 기본값 240MB | **버전/VM 의존** | JDK, VM, Tiered Compilation 설정에 따라 다르므로 운영 JVM에서 `PrintFlagsFinal`로 확인 필요 |
| 10 | Thread Stack 기본값 1024KB | **플랫폼/JDK 의존** | Linux x86_64 HotSpot에서 흔하지만 고정 법칙은 아님 |
| 11 | JVM OOM 시 HeapDumpOnOutOfMemoryError 동작 | **사실** | JVM이 OutOfMemoryError를 처리할 수 있을 때 트리거 |
| 12 | Pod OOM 시 HeapDumpOnOutOfMemoryError 동작 안 함 | **사실** | SIGKILL은 catch 불가, JVM에게 정리 기회 없음 ([HeapHero](https://blog.heaphero.io/oomkilled-vs-java-oom-kubernetes/), [Komodor](https://komodor.com/learn/how-to-fix-oomkilled-exit-code-137/)) |
| 13 | ExitOnOutOfMemoryError 없으면 장애 상태가 지속될 수 있다 | **사실** | JVM OOM은 프로세스 종료를 항상 보장하지 않으며 health check가 통과할 수 있음 |
| 14 | AlwaysPreTouch로 RSS 예측 가능성이 높아진다 | **사실, 단 완전 고정은 아님** | 시작 시 heap page를 미리 touch하지만 native/off-heap 변동은 별도 ([Oracle java options](https://docs.oracle.com/en/java/javase/17/docs/specs/man/java.html), [JVM Field Guide](https://serce.me/posts/2023-02-01-jvm-field-guide-memory)) |
| 15 | MinRamPercentage가 "최소 Heap 크기"를 지정한다 | **부정확** | 실제로는 가용 메모리 ~200MB 미만인 소형 환경에서의 Heap 상한 비율. 대부분 서버에서 무의미 ([Baeldung](https://www.baeldung.com/java-jvm-parameters-rampercentage)) |

---

## Sources

- [Kubernetes - Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes - Assign Memory Resources](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)
- [Kubernetes - Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
- [Sysdig - Kubernetes OOM and CPU Throttling](https://www.sysdig.com/blog/troubleshoot-kubernetes-oom)
- [Komodor - How to Fix OOMKilled (Exit Code 137)](https://komodor.com/learn/how-to-fix-oomkilled-exit-code-137/)
- [HeapHero - OOMKilled vs Java OOM in Kubernetes](https://blog.heaphero.io/oomkilled-vs-java-oom-kubernetes/)
- [Mihai Albert - Out-of-memory in Kubernetes Part 4](https://mihai-albert.com/2022/02/13/out-of-memory-oom-in-kubernetes-part-4-pod-evictions-oom-scenarios-and-flows-leading-to-them/)
- [Baeldung - JVM Parameters: RAMPercentage](https://www.baeldung.com/java-jvm-parameters-rampercentage)
- [DZone - Best Practices: Java Memory for Containers](https://dzone.com/articles/best-practices-java-memory-arguments-for-container)
- [DZone - Difference Between InitialRAMPercentage, MinRAMPercentage, MaxRAMPercentage](https://dzone.com/articles/difference-between-initialrampercentage-minramperc)
- [Oracle - GC Tuning Guide: Other Considerations](https://docs.oracle.com/en/java/javase/11/gctuning/other-considerations.html)
- [Red Hat Developer - Metaspace Setting and Tuning](https://developers.redhat.com/articles/2024/07/19/metaspace-setting-and-tuning-jdk-8-applications-and-outside-containers)
- [stuefe.de - Sizing Metaspace](https://stuefe.de/posts/metaspace/sizing-metaspace/)
- [Baeldung - Introduction to JVM Code Cache](https://www.baeldung.com/jvm-code-cache)
- [SerCe's Blog - JVM Field Guide: Memory](https://serce.me/posts/2023-02-01-jvm-field-guide-memory)
- [Brice Dutheil - Off-Heap Memory Reconnaissance](https://blog.arkey.fr/2020/11/30/off-heap-reconnaissance/)
- [Kubernetes - Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Kubernetes - kube-scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Kubernetes - Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Kubernetes - Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Spring.io - Liveness and Readiness Probes with Spring Boot](https://spring.io/blog/2020/03/25/liveness-and-readiness-probes-with-spring-boot/)
- [Spring Boot - Metrics](https://docs.spring.io/spring-boot/reference/actuator/metrics.html)
- [Google Cloud - Kubernetes best practices: terminating with grace](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-terminating-with-grace)
- [CNCF - Decoding the pod termination lifecycle in Kubernetes](https://www.cncf.io/blog/2024/12/19/decoding-the-pod-termination-lifecycle-in-kubernetes-a-comprehensive-guide/)
- [BlaBlaCar - Warm up the relationship between Java and Kubernetes](https://medium.com/blablacar/warm-up-the-relationship-between-java-and-kubernetes-7fc5741f9a23)
