---
title: "K8S에서 JVM 앱 운영하기 (2) — Graceful Shutdown과 운영"
date: 2026-05-08 10:10:00 +0900
categories: [JVM, Kubernetes]
tags: [kubernetes, jvm, graceful-shutdown, monitoring, hpa, jvm-warmup, micrometer, prometheus]
series: "K8S에서 JVM 앱 운영하기"
---

> 이 글은 **K8S에서 JVM 앱 운영하기** 시리즈의 두 번째 글이다.
> 1. [클러스터 구조와 Probe](/posts/2026-05-08-k8s-jvm-lifecycle)
> 2. **Graceful Shutdown과 운영** ← 현재 글
> 3. [CPU는 느려지고, Memory는 죽는다](/posts/2026-05-08-k8s-jvm-memory)

1편에서 JVM 앱의 배포~Probe 통과까지의 라이프사이클을 다뤘다. 이 글에서는 Pod 종료 시 발생하는 **Race Condition과 Graceful Shutdown**, JVM 모니터링 메트릭, 그리고 HPA 스케일아웃 시 JVM Warmup 문제를 다룬다.

---

## 1. Graceful Shutdown — 종료 시퀀스

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

#### preStop: sleep 10은 왜 필요한가?

```text
preStop이 없을 때:

  SIGTERM 수신 → Spring 종료 시작 → 3초 후 JVM 종료
                                          ↑
  그런데 kube-proxy는 아직 라우팅 규칙 업데이트 중 (5초 소요)
  → 이미 죽은 Pod로 요청이 갈 수 있음

preStop: sleep 10이 있을 때:

  SIGTERM 수신 → preStop: sleep 10 (아무것도 안 하고 10초 대기)
                      ↓
                이 10초 동안 kube-proxy가 라우팅 규칙 업데이트 완료
                → 새 요청이 이 Pod로 오지 않게 됨
                      ↓
                sleep 끝 → 그제서야 Spring 종료 시작
                → 이미 트래픽이 끊긴 상태에서 안전하게 종료
```

kube-proxy가 Endpoint에서 Pod를 빼는 데 수초가 걸리는데, 앱이 그보다 먼저 죽으면 **종료 중인 Pod로 요청이 가는 Race Condition**이 생긴다. `preStop: sleep 10`은 앱 종료를 의도적으로 지연시켜서 그 경합 윈도우를 없애는 완화책이다.

Spring Boot에서 진행 중 요청을 기다리는 종료를 기대하려면 애플리케이션 쪽 graceful shutdown도 켜야 한다. 예를 들어 `server.shutdown=graceful`과 `spring.lifecycle.timeout-per-shutdown-phase`를 termination grace period 안에 들어오도록 맞춘다. SIGTERM을 받는다고 모든 Spring Boot 애플리케이션이 자동으로 진행 중 요청을 끝까지 기다리는 것은 아니다.

```text
타이밍 공식:
  terminationGracePeriodSeconds >= preStop(10s) + Spring shutdown(25s) + 여유(5s) = 40s 이상
  기본값 30초가 부족할 수 있는 이유: preStop(10s) + Spring shutdown(25s) + 여유(5s) = 40s > 30s → 유예 시간 초과 시 SIGKILL 가능
```

> **근거**: Google Cloud Blog — *"Pod 종료 시 SIGTERM과 Endpoint 제거는 동시에 시작된다. preStop hook으로 앱이 Endpoint 제거보다 먼저 죽는 것을 방지해야 한다."* CNCF Blog — *"terminationGracePeriodSeconds는 전체 종료 과정(preStop + SIGTERM 처리)을 포함하는 유예 시간이다."* ([Google Cloud - Terminating with grace](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-terminating-with-grace), [CNCF - Pod termination lifecycle](https://www.cncf.io/blog/2024/12/19/decoding-the-pod-termination-lifecycle-in-kubernetes-a-comprehensive-guide/))

---

## 2. 모니터링 — JVM Metrics

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

## 3. HPA와 JVM Warmup 문제

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

다음 글에서는 이 시리즈의 핵심 주제인 **CPU와 Memory가 왜 다르게 동작하는가**, JVM 메모리 구조, OOM의 두 가지 유형, 리소스 설정 전략을 다룬다.
