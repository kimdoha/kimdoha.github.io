---
title: "OpenTelemetry로 보는 분산 추적 — 개념 정리와 Pinpoint와의 비교"
date: 2026-06-04 10:00:00 +0900
categories: [Observability, OpenTelemetry]
tags: [opentelemetry, otel, otlp, distributed-tracing, pinpoint, observability]
toc: true
---

마이크로서비스 환경에서 한 요청이 여러 서비스를 거치는 일은 일상이 되었다. 이때 "어느 구간에서 느려졌나", "어디서 에러가 났나"를 추적하려면 분산 추적이 필요하다. OpenTelemetry는 이런 분산 추적과 메트릭, 로그를 **벤더 중립적인 표준**으로 통합하려는 프로젝트다. 이 글에서는 OpenTelemetry의 핵심 개념을 정리하고, 국내 환경에서 흔히 쓰이는 Pinpoint와 어떤 차이가 있는지 비교한다.

---

## 1. 분산 추적이 풀려는 문제

```
Client ──► API Gateway ──► Service A ──► Service B ──► DB
                                  └────► Service C ──► Cache
```

요청 하나가 여러 서비스를 거치는 구조에서는 다음 질문에 답하기 어렵다.

- 응답이 1.2초 걸렸는데, **어느 서비스가 그중 800ms를 썼나?**
- DB 쿼리 한 건이 느려졌는데, **어떤 사용자 요청에서 호출된 쿼리인가?**
- 5xx 에러가 발생했는데, **호출 체인의 어디에서 처음 발생했나?**

분산 추적은 한 요청을 식별 가능한 ID로 묶어, 각 서비스가 처리한 구간(span)을 시간 순서로 재구성한다. 이를 통해 위 질문에 답할 수 있다.

---

## 2. OpenTelemetry란

[OpenTelemetry(OTel)](https://opentelemetry.io/)는 분산 추적, 메트릭, 로그를 **단일 SDK와 표준 프로토콜(OTLP)**로 다루는 CNCF 프로젝트다. 2019년 5월 OpenTracing(추적)과 OpenCensus(추적+메트릭)가 통합되며 출범했고, 2021년 8월 CNCF Incubating 단계에 진입한 뒤 [2026년 5월 21일 CNCF에서 graduation이 발표](https://www.cncf.io/announcements/2026/05/21/cloud-native-computing-foundation-announces-opentelemetrys-graduation-solidifying-status-as-the-de-facto-observability-standard/)되어 관측성 분야의 사실상 표준으로 자리잡았다.

핵심 가치는 두 가지다.

- **벤더 중립**: Jaeger, Zipkin, Tempo, Datadog, New Relic 등 어떤 백엔드에도 같은 SDK로 데이터를 보낼 수 있다. 백엔드를 바꿀 때 코드 수정이 필요 없다.
- **표준화**: OTLP라는 단일 프로토콜로 추적/메트릭/로그를 한 번에 전송한다.

세 신호(Trace / Metric / Log)는 같은 속도로 안정화되지는 않았다. [공식 Specification Status](https://opentelemetry.io/docs/specs/status/) 기준으로 다음과 같이 단계별로 GA가 진행됐다.

| 신호 | Stable 시점 |
|---|---|
| **Tracing** | [Specification 1.0 — 2021년 2월](https://medium.com/opentelemetry/opentelemetry-specification-v1-0-0-tracing-edition-72dd08936978) |
| **Metrics** | 언어별 2023년부터 점진 stable (Java, Go 등 선도) |
| **Logs** | 데이터 모델·Bridge API·SDK·Protocol은 stable. 단, 로그 appender(언어별 로그 라이브러리 ↔ OTel 연결 구현체)는 여러 언어에서 개발 중 |

새 프로젝트에 도입할 때는 Tracing은 안심하고 채택해도 되지만, Logs는 도입 시점의 언어별 안정성을 확인해야 한다.

---

## 3. 핵심 개념

### 3.1 Trace 와 Span

- **Trace**: 한 요청의 전체 처리 흐름. 고유한 `trace_id`로 식별된다.
- **Span**: Trace를 구성하는 단위 작업. HTTP 요청 한 건, DB 쿼리 한 건, 외부 API 호출 한 건이 모두 Span이 될 수 있다.

```
Trace (trace_id = abc123)
│
├── Span: GET /products/{id}             (service-gateway)
│   ├── Span: ProductService.getDetail   (service-product)
│   │   ├── Span: SELECT FROM product    (DB)
│   │   └── Span: GET cache:product:42   (Redis)
│   └── Span: InventoryService.getStock  (service-inventory)
│       └── Span: SELECT FROM stock      (DB)
```

각 Span은 `start_time`, `end_time`, `parent_span_id`, `attributes`(태그)를 가진다. 부모-자식 관계로 트리를 이루며, 이 트리가 곧 한 요청의 분산 처리 흐름이다.

### 3.2 Context Propagation

서비스 경계를 넘어갈 때 `trace_id`와 `span_id`를 전파해야 호출 체인이 연결된다. OpenTelemetry는 [W3C Trace Context](https://www.w3.org/TR/trace-context/) 표준을 따라 HTTP 헤더로 전파한다.

```
traceparent: 00-abc123def456...-001100110011-01
                 ↑                ↑           ↑
                 trace_id         span_id    flags
```

서비스 A가 서비스 B를 HTTP로 호출할 때 이 헤더를 함께 전송하면, 서비스 B는 자신의 Span을 만들 때 `parent_span_id`로 A의 span_id를 사용한다. 결과적으로 트리가 연결된다.

### 3.3 Baggage

Trace Context와 별개로, 서비스 간에 전파하고 싶은 **cross-cutting 메타데이터**가 있다. 예를 들어 `user_id`, `tenant_id`, `experiment_group` 같은 값이다. OpenTelemetry는 이를 **Baggage**라는 별도 헤더(`baggage: user_id=42,tenant_id=mall-123`)로 전파한다.

Baggage는 모든 다운스트림 서비스에서 읽을 수 있어, 로그/메트릭에 일관된 라벨을 붙이는 용도로 유용하다. 다만 [공식 문서](https://opentelemetry.io/docs/concepts/signals/baggage/)는 Baggage가 무결성 검증 없이 외부 API까지 전파될 수 있어 민감 정보를 담거나 보안 의사결정에 사용해서는 안 된다고 경고한다. 비즈니스 분기·라우팅에 쓰고 싶다면 신뢰 경계 안쪽에서 매우 제한적으로만 활용하는 것이 안전하다.

---

## 4. Instrumentation: Auto vs Manual

애플리케이션에 추적 코드를 심는 방식은 두 가지다.

### 4.1 Auto-instrumentation

Java 기준 `opentelemetry-javaagent.jar`를 JVM에 attach하면, 별도 코드 변경 없이 다음 항목들이 자동으로 추적된다.

- HTTP server/client (Spring MVC, WebFlux, OkHttp, Apache HttpClient 등)
- JDBC, Redis, MongoDB
- Kafka producer/consumer
- gRPC

```bash
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=my-service \
     -Dotel.exporter.otlp.endpoint=http://collector:4317 \
     -jar app.jar
```

동작 방식은 Java Instrumentation API의 `premain` 훅으로 진입한 뒤 [ByteBuddy](https://bytebuddy.net/) 기반으로 대상 클래스의 바이트코드를 런타임에 변환하는 구조다. 소스 코드 수정 없이 트레이스가 수집되는 대신, 라이브러리 버전마다 호환 가능 범위가 다르므로 [지원 라이브러리 매트릭스](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/docs/supported-libraries.md)를 사전에 확인해야 한다.

### 4.2 Manual instrumentation

비즈니스 로직 내부의 특정 구간을 명시적으로 추적하고 싶을 때 사용한다.

```kotlin
val tracer = GlobalOpenTelemetry.getTracer("product-service")

fun calculateDiscount(productNo: Long): BigDecimal {
    val span = tracer.spanBuilder("calculateDiscount")
        .setAttribute("product.no", productNo)
        .startSpan()
    try {
        return span.makeCurrent().use {
            // 비즈니스 로직
        }
    } finally {
        span.end()
    }
}
```

실무에서는 Auto-instrumentation을 기본으로 깔고, 도메인 로직 중 추적이 필요한 부분만 Manual로 보강하는 패턴이 일반적이다.

---

## 5. 데이터 수집 파이프라인

OpenTelemetry는 SDK에서 직접 백엔드로 보내지 않고, 중간에 **Collector**를 두는 구조를 권장한다.

```
Application (SDK)
    │ OTLP/gRPC
    ▼
OpenTelemetry Collector
  ├─ receivers   : OTLP, Jaeger, Zipkin, Prometheus, ...
  ├─ processors  : batch, attributes, sampling, filter
  └─ exporters   : Jaeger, Zipkin, Tempo, Datadog, ...
    │
    ▼
Backend (Jaeger / Tempo / Datadog / ...)
```

Collector의 이점은 다음과 같다.

- **샘플링/필터링을 한 곳에서**: 트래픽이 큰 환경에서 모든 trace를 저장하면 비용이 폭증한다. Tail-based sampling으로 에러가 난 trace만 저장하는 등의 정책을 Collector 단에서 적용한다.
- **백엔드 교체가 쉬움**: 애플리케이션 설정은 그대로 두고 Collector의 exporter만 바꾸면 백엔드 마이그레이션이 가능하다.
- **포맷 변환**: Jaeger/Zipkin 같은 레거시 포맷을 OTLP로, 또는 그 반대로 변환할 수 있다.

### 5.1 OTLP 프로토콜

[OTLP(OpenTelemetry Protocol)](https://opentelemetry.io/docs/specs/otlp/)는 SDK/Agent와 Collector/백엔드 사이에서 텔레메트리 데이터를 교환하는 표준 프로토콜이다. Trace·Metric·Log 세 신호를 같은 프로토콜로 전송할 수 있다는 점이 핵심이며, [opentelemetry-proto v1.0.0이 2023년 7월에 stable로 릴리스](https://github.com/open-telemetry/opentelemetry-proto/releases/tag/v1.0.0)되어 운영 환경 도입에 안정적인 기반이 마련됐다.

**전송 방식**

OTLP는 두 가지 wire format을 정의한다.

| 전송 | 기본 포트 | Content-Type | 인코딩 |
|---|---|---|---|
| OTLP/gRPC | 4317 | `application/grpc` | Protocol Buffers (binary) |
| OTLP/HTTP | 4318 | `application/x-protobuf` 또는 `application/json` | Protocol Buffers 또는 JSON |

OTLP/gRPC는 HTTP/2 multiplexing과 효율적 binary 인코딩 덕분에 고성능 내부망 전송에서 자주 선택된다. OTLP/HTTP는 방화벽이 gRPC를 차단하는 환경이나 브라우저·Serverless 환경에서 활용된다. [opentelemetry-proto](https://github.com/open-telemetry/opentelemetry-proto) 리포지토리에 `.proto` 정의가 공개되어 있어 자체 SDK 구현도 가능하다.

**서비스 엔드포인트**

OTLP/HTTP는 신호별로 분리된 경로를 사용한다.

```
POST /v1/traces      # Trace 데이터
POST /v1/metrics     # Metric 데이터
POST /v1/logs        # Log 데이터
```

OTLP/gRPC는 신호별로 별도 service가 정의되어 있다 — `opentelemetry.proto.collector.{trace,metrics,logs}.v1.{Trace,Metrics,Logs}Service` 의 `Export` RPC.

**압축**

[명세](https://opentelemetry.io/docs/specs/otlp/)는 OTLP 서버가 압축 없음(`none`)과 `gzip`을 반드시 지원해야 한다고 규정한다. 그 외 알고리즘은 구현체별로 선택적으로 지원될 수 있다. 대용량 trace 전송 시 `gzip`만 적용해도 네트워크 비용을 크게 줄일 수 있다.

**Partial Success 응답**

OTLP는 receiver가 일부 레코드만 처리에 실패한 경우를 표현하는 `partial_success` 응답을 정의한다. 예를 들어 1,000개 span 중 30개만 schema 오류로 거부됐다면, receiver는 거부된 개수와 메시지를 함께 돌려준다.

```protobuf
message ExportTraceServiceResponse {
  ExportTracePartialSuccess partial_success = 1;
}

message ExportTracePartialSuccess {
  int64 rejected_spans = 1;
  string error_message = 2;
}
```

단, [OTLP 명세](https://opentelemetry.io/docs/specs/otlp/)는 sender가 `partial_success`를 받았을 때 해당 요청을 **재시도해서는 안 된다**고 명시한다. 거부된 레코드는 영구 실패로 처리하고 로깅·알림으로만 남기되, 성공적으로 처리된 나머지는 그대로 둔다. 이 응답 덕분에 schema 오류 같은 부분 실패를 명확히 가시화하면서도 중복 전송을 피할 수 있다.

**Retry & 상태 코드**

[OTLP 명세](https://opentelemetry.io/docs/specs/otlp/#failures-and-retries)는 retryable/non-retryable 오류를 명확히 구분한다.

- **Retryable** — HTTP 429 (Too Many Requests), 502/503/504, gRPC `UNAVAILABLE`·`RESOURCE_EXHAUSTED` 등. 지수 백오프로 재시도하되 sender는 `Retry-After`(HTTP) 또는 gRPC trailers의 `grpc-retry-pushback-ms`를 존중해 backpressure에 대응한다.
- **Non-retryable** — HTTP 400 (Bad Request), 401 (Unauthorized), gRPC `INVALID_ARGUMENT`·`UNAUTHENTICATED` 등. 재시도하지 않고 즉시 실패 처리해야 한다.

이 규약 덕분에 SDK·Collector·백엔드가 서로 다른 벤더 구현이어도 retry 동작이 호환된다.

---

### 5.2 Collector 배포 모드

같은 Collector 바이너리라도 어디에 배치하느냐에 따라 운영 특성이 달라진다. [공식 문서](https://opentelemetry.io/docs/collector/deploy/)는 세 가지 배포 패턴을 제시한다.

**Agent 패턴**

```
[Application] ──► [Collector (sidecar/DaemonSet)] ──► Backend
```

각 워크로드 옆에 Collector를 사이드카 또는 DaemonSet으로 배치한다. 애플리케이션과 Collector가 같은 호스트/Pod 안에 있어 네트워크 의존도가 낮고, 호스트 레벨 메트릭(CPU, 메모리, 디스크) 수집에 유리하다. 다만 워크로드 수만큼 Collector 인스턴스가 늘어나 리소스 부담이 누적된다.

**Gateway 패턴**

```
[Application 1] ──┐
[Application 2] ──┼──► [Collector (Deployment)] ──► Backend
[Application 3] ──┘
```

독립 서비스(예: Kubernetes Deployment)로 Collector를 중앙에 두고, 모든 애플리케이션이 OTLP 엔드포인트로 데이터를 보낸다. 중앙에서 배치, enrichment, **tail-based sampling**, rate limiting 같은 정책을 적용하기 좋다. 다만 Collector가 단일 장애점이 되지 않도록 다중 replica + Load Balancer 구성이 필요하다.

**Agent + Gateway 결합**

```
[Application] ──► [Agent (sidecar)] ──► [Gateway (Deployment)] ──► Backend
```

각 호스트의 Agent가 1차 수집(호스트 메트릭, 로컬 enrichment)을 담당하고, 중앙 Gateway가 2차 처리(샘플링, 백엔드 분기)를 담당한다. 운영 규모가 커진 환경에서 가장 자주 채택되는 모델이다. 트레이드오프는 운영 복잡도 증가다.

도입 초기에는 Gateway 한 형태로 시작해 운영 부담을 줄이고, 트래픽이 커지거나 호스트 레벨 수집이 필요해진 시점에 Agent를 도입하는 단계적 접근이 안전하다.

### 5.3 파이프라인 구성: Receivers / Processors / Exporters

Collector 내부는 **Receivers(수신) → Processors(가공) → Exporters(전송)** 세 단계로 조립된다. 각 단계의 역할이 명확히 분리되어 있어 Lego 블록처럼 필요한 컴포넌트만 골라 끼울 수 있다.

**Receivers — 입구**

외부에서 telemetry 데이터를 받아들이는 컴포넌트. OTLP가 표준이지만 레거시 호환을 위해 Jaeger, Zipkin, Prometheus scrape, Kafka, filelog 등 다양한 receiver가 제공된다. 한 Collector에 여러 receiver를 동시에 등록할 수 있어, OTLP·Jaeger·Prometheus를 동시에 받는 마이그레이션 단계에서도 유용하다.

**Processors — 중간 가공·필터링**

받은 데이터를 가공·필터·배치하는 파이프라인이며, 정의 **순서가 곧 실행 순서**다. 자주 쓰는 processor는 다음과 같다.

| Processor | 역할 |
|---|---|
| `memory_limiter` | OOM 방지 — 임계치 초과 시 새 데이터 거부. 보통 **최상단**에 배치 |
| `batch` | 여러 record를 묶어 export — 네트워크 비용·백엔드 부하 감소. 보통 **최하단**에 배치 |
| `attributes` | attribute 추가·삭제·마스킹 (민감정보 필터링) |
| `resource` | `service.name`·`deployment.environment` 등 resource attribute 부여 |
| `tail_sampling` | 전체 trace 완료 후 샘플링 결정 (에러난 trace만 저장 등) |
| `filter` | 헬스체크 trace 같은 무가치 데이터 제외 |

**Exporters — 출구**

처리된 데이터를 백엔드로 전송한다. 백엔드별로 전용 exporter를 선택하며, Grafana 관측성 스택 기준 매핑은 다음과 같다.

| 신호 | Exporter | 대상 백엔드 |
|---|---|---|
| Trace | `otlp` / `otlphttp` | Grafana **Tempo** (OTLP 네이티브 수신) |
| Metric | `prometheusremotewrite` | Grafana **Mimir** (Prometheus 호환) |
| Log | `otlphttp` | Grafana **Loki** (OTLP ingest endpoint) |

> Loki는 [공식 가이드](https://grafana.com/docs/enterprise-logs/latest/send-data/otel/)에서 OTLP ingest endpoint(`/otlp/v1/logs`)를 안내한다. 과거 OTel Collector에 있던 전용 `loki` exporter는 [현재 contrib 컴포넌트 목록](https://opentelemetry.io/docs/collector/components/exporter/)에서 빠졌으니, 신규 구성에서는 `otlphttp` exporter로 보내는 것이 표준이다.

Grafana Cloud을 사용한다면 위 3개를 별도 운영하지 않고 단일 OTLP endpoint로 `otlphttp` exporter 하나만 둬도 된다.

> **⚠ 자주 헷갈리는 부분 — Grafana 자체는 exporter 대상이 아니다.**
>
> Grafana는 데이터를 *받는* 백엔드가 아니라, Tempo/Mimir/Loki 같은 백엔드를 `data source`로 등록해 **조회·시각화**하는 프론트엔드다. OTel Collector exporter는 항상 스토리지·쿼리 엔진(Tempo/Mimir/Loki/Prometheus/Datadog/Jaeger 등)을 가리키고, Grafana는 그 뒤에서 PromQL/LogQL/TraceQL로 읽어올 뿐이다.
>
> ```
> [Collector] ──exporter──► [Tempo/Mimir/Loki]  ◄── data source ── [Grafana]
>                            (저장·쿼리 엔진)                       (대시보드·UI)
> ```
>
> 따라서 exporter 결정은 "어느 백엔드에 저장할지"의 문제이고, Grafana 연결은 별도 단계(data source 등록)다.
>
> | 환경 | Exporter 설정 |
> |---|---|
> | Grafana + 셀프호스팅 Tempo/Mimir/Loki | 각각에 전용 exporter (`otlp/tempo`, `prometheusremotewrite/mimir`, `otlphttp/loki`) |
> | Grafana Cloud (managed 번들) | `otlphttp/grafanacloud` 단일 endpoint |
> | Grafana + Prometheus만 있음 | metrics만 `prometheusremotewrite`, traces/logs는 별도 백엔드 필요 |

**조립 예시 (Grafana 스택)**

세 컴포넌트는 **신호별 pipeline**으로 묶어 `service.pipelines` 아래 정의한다.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  memory_limiter:
    limit_mib: 512
    check_interval: 1s
  batch:
    timeout: 10s
    send_batch_size: 1024
  attributes/mask:
    actions:
      - key: http.request.header.authorization
        action: delete

exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls: { insecure: true }
  prometheusremotewrite/mimir:
    endpoint: http://mimir:9009/api/v1/push
  otlphttp/loki:
    endpoint: http://loki:3100/otlp

service:
  pipelines:
    traces:
      receivers:  [otlp]
      processors: [memory_limiter, attributes/mask, batch]
      exporters:  [otlp/tempo]
    metrics:
      receivers:  [otlp]
      processors: [memory_limiter, batch]
      exporters:  [prometheusremotewrite/mimir]
    logs:
      receivers:  [otlp]
      processors: [memory_limiter, batch]
      exporters:  [otlphttp/loki]
```

**데이터 흐름**

```
[App SDK] ─OTLP gRPC/HTTP─► ┌──────── OTel Collector ────────┐
                            │ Receivers  Processors  Exporters│
                            │  ┌─────┐  ┌──────────┐ ┌──────┐ │
                            │  │ otlp│─►│memlimiter│►│tempo │─┼──► Tempo   (Trace)
                            │  └─────┘  │attributes│ ├──────┤ │
                            │           │tailsampl │►│mimir │─┼──► Mimir   (Metric)
                            │           │batch     │ ├──────┤ │
                            │           └──────────┘►│loki  │─┼──► Loki    (Log)
                            │                        └──────┘ │
                            └─────────────────────────────────┘
                                                               │
                                                               ▼
                                                          [Grafana]
                                                          대시보드·Explore·알림
```

**실무 팁**

- `memory_limiter`는 항상 파이프라인 **최상단**에 배치한다. OOM이 가장 흔한 Collector 장애 원인이라 backpressure를 가장 먼저 걸어야 한다.
- `batch`는 **최하단**에 배치한다. export 직전에 묶어 보내야 압축·전송 효율이 가장 높다.
- `tail_sampling`은 메모리 부담이 크고, 같은 `trace_id`의 모든 span이 같은 Collector 인스턴스로 모여야 동작한다. Gateway 앞단에 trace_id 기반 라우팅 LB를 두는 구성이 필요하다.
- 같은 신호를 두 exporter에 동시에 export할 수 있다(예: `[otlp/tempo, otlp/datadog]`). 백엔드 마이그레이션 시 dual-write로 활용하면 무중단 전환이 가능하다.

---

## 6. Pinpoint와의 비교

국내 환경에서는 [Pinpoint](https://pinpoint-apm.github.io/pinpoint/)가 널리 쓰인다. 2012년 NAVER에서 개발이 시작되어 2015년 1월 오픈소스로 공개된 APM 도구로, OpenTelemetry보다 먼저 자리잡은 분산 추적 솔루션이다. 두 도구는 목표는 비슷하지만 접근 방식이 다르다.

| 항목 | OpenTelemetry | Pinpoint |
|---|---|---|
| **표준화** | W3C Trace Context, OTLP 표준 채택 | 자체 프로토콜 |
| **데이터 종류** | Trace + Metric + Log 통합 | 주로 Trace + 일부 Metric |
| **벤더 종속성** | 벤더 중립 (백엔드 교체 가능) | Pinpoint Web UI + HBase 등 자체 스택 |
| **Instrumentation** | Java Agent (Auto) + SDK (Manual) | Java Agent (Auto) 위주 |
| **언어 지원** | Java/Kotlin, Go, Python, JS/TS, Rust, .NET 등 다수 | Java(메인), PHP, Python |
| **운영 부담** | Collector + Backend 별도 구성 필요 | Pinpoint Web/Collector/Storage 중심의 통합 스택 (HBase, 메트릭은 Pinot/Kafka 등 추가 의존) |
| **국내 자료** | 영문 중심, 한글 자료 증가 추세 | 한글 문서/사례 풍부 |

도입 측면에서 보면, **Pinpoint는 "켜면 바로 보임"의 즉시성이 강점**이고, **OpenTelemetry는 "어디로든 보낼 수 있음"의 유연성이 강점**이다. 새로 시작하는 프로젝트나 다중 언어/다중 백엔드 환경이라면 OpenTelemetry가 유리하고, 단일 JVM 환경에서 빠르게 분산 추적을 도입하고 싶다면 Pinpoint가 무난하다.

두 도구를 **병행 운영**하는 것도 가능하다. 실제로 OpenTelemetry Agent와 Pinpoint Agent를 함께 attach하면 양쪽에 trace가 동시에 들어간다. 점진적 마이그레이션이나 백엔드 비교 평가 단계에서 유용한 패턴이다.

---

## 7. 도입 시 고려할 점

OpenTelemetry를 새로 도입한다면 다음을 미리 정해두는 편이 안전하다.

- **샘플링 전략**: 모든 trace를 저장하면 스토리지 비용이 빠르게 커진다. Head-based(요청 진입 시점에 결정) 또는 Tail-based(전체 trace 완료 후 결정) 중 정책을 선택한다.
- **서비스 이름 규칙**: `otel.service.name`은 백엔드에서 trace를 묶는 키가 된다. K8s namespace/deployment 명명 규칙과 일관되게 설정한다.
- **민감 정보 필터링**: HTTP path, query, header가 그대로 attribute로 들어갈 수 있다. Collector의 `attributes` processor로 마스킹 처리를 한다.
- **Resource attribute**: `service.name`, `service.version`, `deployment.environment` 같은 표준 attribute는 SDK 초기화 시 명시한다. 백엔드에서 grouping 키로 사용된다.
- **Span 한도**: 한 trace가 너무 많은 span을 만들면 backend가 trim한다. 루프 안에서 매번 span을 열지 않도록 한다.

---

## 마치며

분산 추적은 마이크로서비스 환경의 디버깅 비용을 줄이는 가장 효과적인 투자 중 하나다. OpenTelemetry는 그 영역의 표준으로 자리잡고 있고, Pinpoint 같은 기존 도구와 병행할 수도 있다. 핵심은 **trace_id가 서비스 경계를 넘어 일관되게 흐르도록 만드는 것**이고, OTel은 그 일관성을 표준 프로토콜로 보장한다는 점이 가장 큰 강점이다.

### 운영 환경 샘플링 정책

Pinpoint는 보통 100% 샘플링으로 운영되는 경우가 많지만(샘플링 비율 자체는 조절 가능하다), OTel로 전환하면서 환경별 샘플링 정책을 다음과 같이 잡을 수 있다. 핵심 아이디어는 **"에러와 느린 요청은 빠짐없이, 정상 트래픽은 가볍게"** 다.

| 구분 | 정책 |
|---|---|
| **알파(Alpha)** | 100% 샘플링 |
| **헬스체크** | 수집 제외 |
| **HTTP 4xx / 5xx** | 100% 수집 |
| **응답 500ms 이상** | 100% 수집 |
| **그 외 정상 요청** | 1% 샘플링 |

리얼 환경에서는 풀 샘플링과 달리 에러·지연만 100% 모으고 정상 건은 1%로 줄여 인프라 부하와 스토리지 비용을 함께 잡는 방식이다.

최근 사내에서 Pinpoint에서 OpenTelemetry로 이관하여, 이번 글에서는 OpenTelemetry의 개념과 운영 시 고려할 점을 전반적으로 정리해보았다.