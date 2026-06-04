---
title: "K8S에서 JVM 앱 운영하기 (3) — CPU Throttling과 메모리 한계 동작"
date: 2026-05-08 10:20:00 +0900
categories: [JVM, Kubernetes]
tags: [kubernetes, jvm, oom, pod-resources, memory, cpu-throttling, qos]
series: "K8S에서 JVM 앱 운영하기"
---

> 이 글은 **K8S에서 JVM 앱 운영하기** 시리즈의 세 번째 글이다.
> 1. [클러스터 구조와 Probe](/posts/2026-05-08-k8s-jvm-lifecycle)
> 2. [Graceful Shutdown과 운영](/posts/2026-05-08-k8s-jvm-operations)
> 3. **CPU는 느려지고, Memory는 죽는다** ← 현재 글

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

---

## 1. JVM OOM vs Pod OOM

Memory 초과 시, **누가 먼저 감지하느냐**에 따라 증상과 대응이 완전히 다르다.

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

## 2. JVM 메모리는 Heap만이 아니다

컨테이너의 memory limit은 JVM heap 크기만 보는 것이 아니라 **컨테이너 cgroup에 잡히는 전체 메모리 사용량**을 기준으로 적용된다. 따라서 `-Xmx`만 Pod memory limit에 맞추면 안전하지 않다. HotSpot JVM의 Native Memory Tracking(NMT) 기준으로 보면 JVM 프로세스의 메모리는 대략 다음처럼 나눠서 봐야 한다.

```text
┌──────────────────────────── Container / Pod memory limit ────────────────────────────┐
│                                                                                      │
│  ┌────────────────────────────── JVM Process RSS / cgroup memory ─────────────────┐  │
│  │                                                                                │  │
│  │  ┌──────────────────────── Java Heap ────────────────────────┐                 │  │
│  │  │  Young / Old Gen                                           │                 │  │
│  │  │  Java objects, arrays                                      │                 │  │
│  │  │  -Xmx, MaxRAMPercentage의 주요 대상                         │                 │  │
│  │  └────────────────────────────────────────────────────────────┘                 │  │
│  │                                                                                │  │
│  │  ┌──────────────────────── Non-Heap / JVM Native ───────────────────────────┐   │  │
│  │  │  Class / Metaspace       : class metadata                                │   │  │
│  │  │  Thread                  : Java/native thread stacks, thread structures   │   │  │
│  │  │  Code                    : JIT compiled code, Code Cache                  │   │  │
│  │  │  GC                      : GC data structures, remembered/card tables     │   │  │
│  │  │  Compiler                : JIT compiler working memory                    │   │  │
│  │  │  Symbol / String tables  : VM symbols, interned string related metadata   │   │  │
│  │  │  Internal / Arena / NMT  : VM internal allocations, tracking overhead      │   │  │
│  │  └───────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                                                                │  │
│  │  ┌────────────────────── Off-Heap / OS-visible memory ───────────────────────┐  │  │
│  │  │  DirectByteBuffer / Netty direct memory                                    │  │  │
│  │  │  JNI / native library allocations                                          │  │  │
│  │  │  mmap files, shared libraries, libc malloc arenas                          │  │  │
│  │  │  tmpfs emptyDir, page cache 등 컨테이너 메모리에 잡힐 수 있는 영역          │  │  │
│  │  └────────────────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                                │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘

핵심: Pod OOM은 Java Heap OOM이 아니라 cgroup memory limit 초과다.
```

위 그림에서 `Java Heap`은 보통 가장 큰 영역이지만 전부가 아니다. `Metaspace`, thread stack, code cache, GC 구조체, direct buffer, native library allocation까지 합쳐서 컨테이너 limit을 넘으면 JVM이 `OutOfMemoryError`를 던지기 전에 커널이 프로세스를 SIGKILL할 수 있다. 그래서 Kubernetes에서 JVM 앱을 운영할 때는 다음처럼 잡아야 한다.

```text
Pod memory limit
  >
    Java Heap(-Xmx)
  + Metaspace / Code Cache / GC native memory
  + Thread stacks
  + Direct / off-heap memory
  + native library / mmap / page cache / tmpfs 사용량
  + 운영 여유분
```

> **근거**: Oracle HotSpot Native Memory Tracking 문서는 `Java Heap`, `Class`, `Thread`, `Code`, `GC`, `Compiler`, `Internal`, `Symbol`, `Native Memory Tracking` 등 JVM 내부 메모리 카테고리를 구분한다. Oracle Java 명령 문서도 NMT가 Java heap, class, code, thread 같은 JVM subsystem 단위 메모리를 추적한다고 설명한다. Kubernetes 공식 문서는 컨테이너가 memory limit을 초과하면 종료 대상이 될 수 있고, memory limit 적용은 반응적이라고 설명한다. ([Oracle - NMT Memory Categories](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr022.html), [Oracle Java Command - NativeMemoryTracking](https://docs.oracle.com/en/java/javase/22/docs/specs/man/java.html), [Kubernetes Docs - Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/))

---

## 3. CPU와 Memory는 왜 다르게 동작하는가

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

---

## 4. 리소스 설정 전략

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

---

## 5. QoS (Quality of Service)

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