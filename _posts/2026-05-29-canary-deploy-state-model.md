---
title: "ArgoCD + Helm 카나리 배포의 상태 모델과 운영 주의점"
date: 2026-05-29 10:00:00 +0900
categories: [DevOps, Kubernetes]
tags: [argocd, helm, canary, deployment, kubernetes, gitops]
toc: true
---

ArgoCD와 Helm으로 구성된 환경에서 카나리 배포 단계를 `deploy.canary.state` 단일 필드로 표현하는 모델을 다룬다. 이 필드 값을 바꾸는 것이 곧 배포 단계 전이이고, ArgoCD가 그 결과를 Kubernetes 리소스에 반영한다. 마지막 섹션에서는 운영 중 발견한 상태 전이 관련 주의점을 정리한다.

---

## 1. 배포 파이프라인

```
PR (chart values 변경)
   ↓
apps-manifest 레포 (GitHub)
   ↓
ArgoCD Auto Sync
   ↓
Helm Chart 렌더링
   ↓
Kubernetes Deployment (main / sub)
```

- **apps-manifest** 레포는 stage별 Helm values 파일만 보관한다. 경로 규칙은 `chart/{stage}/backend/{namespace}/{app}.yaml`이다.
- PR 머지가 곧 배포 트리거다. ArgoCD가 변경을 감지해 자동으로 동기화하므로 kubectl로 직접 조작하지 않는다.
- Helm chart 템플릿은 별도 chart 레포에 정의되어 있고, values의 `deploy.canary.state` 값을 읽어 deployment 매니페스트를 main / sub 두 벌로 렌더링한다.

values 파일의 핵심 부분은 다음과 같다.

```yaml
deploy:
  strategy: canary
  canary:
    state: finish              # 4가지 상태 중 하나: canary / finish / rollback / deploy_force
    canaryReplicas: 1
    version:
      normal: gateway-1.0.212  # main deployment 이미지 태그
      canary: gateway-1.0.210  # sub deployment 이미지 태그
replicas: 4                    # main deployment replicas
```

---

## 2. 두 개의 Deployment

Helm chart는 한 애플리케이션을 두 개의 Deployment로 렌더링한다.

| Deployment | Label | 역할 |
|---|---|---|
| `{app}-main` | `canary=normal` | 안정 버전. 평상 시 전체 트래픽을 받는다 |
| `{app}-sub` | `canary=canary` | 카나리 버전. 검증 중인 신버전 |

Service 셀렉터는 두 Deployment의 Pod를 모두 endpoint로 묶는다. 별도 가중치 설정이 없는 구조라 트래픽은 endpoint 단위로 분배되며, 메인 4대와 카나리 1대 구성이면 신버전이 받는 트래픽이 장기 평균상 약 20%에 근접한다. 실제 비율은 kube-proxy 모드, connection reuse(HTTP/2·gRPC), session affinity, readiness 등에 따라 달라진다.

세밀한 가중치(1%, 5% 단위) 분배가 필요하다면 Argo Rollouts + Istio/NGINX 조합이 일반적이다. 이 글에서 다루는 구조는 그보다 단순한 모델이다.

---

## 3. 4가지 상태

`deploy.canary.state` 값을 바꾸는 PR이 곧 상태 전이를 일으킨다.

```
┌────────────────────┐    CANARY     ┌────────────────────┐    FINISH    ┌────────────────────┐
│ 시작 / ROLLBACK    │ ────────────► │  CANARY 진행 중    │ ───────────► │ FINISH (정상 운영) │
│ main : N N N       │               │  main : N N N      │              │ main : C C C       │
│ sub  :  .          │ ◄──────────── │  sub  :  C         │              │ sub  :  .          │
└────────────────────┘   ROLLBACK    └────────────────────┘              └────────────────────┘
         │                                                                          ▲
         │                                                                          │
         └─────────────────────────── DEPLOY_FORCE ─────────────────────────────────┘

범례
  N = 구버전 인스턴스       C = 신버전 인스턴스       . = 인스턴스 없음
  main : Deployment with label canary=normal
  sub  : Deployment with label canary=canary
```

### 3.1 canary — 카나리 시작

```yaml
canary:
  state: canary
  canaryReplicas: 1
  version:
    normal: gateway-1.0.212
    canary: gateway-1.0.213   # 검증할 신버전
```

- `main`: 기존 버전(`normal`) × `replicas(4)` 유지
- `sub`: 신버전(`canary`) × `canaryReplicas(1)` 신규 기동

이 단계에서 신버전이 일부 트래픽을 받는다. 이상 신호 유무를 메트릭과 로그로 확인한 뒤 다음 상태를 결정한다.

### 3.2 finish — 카나리 승격

```yaml
canary:
  state: finish
  version:
    normal: gateway-1.0.213   # 신버전을 normal로 승격
    canary: gateway-1.0.212   # 직전 normal을 canary 슬롯에 기록용으로 남김
```

- `main`: 신버전 × `replicas(4)`로 전체 교체
- `sub`: 0대

검증을 마친 신버전을 전체에 적용한다. 정상 운영 상태는 항상 `state: finish`다.

### 3.3 rollback — 카나리 철회

```yaml
canary:
  state: rollback
  # version 변경 없음
```

- `main`: 기존 버전 유지
- `sub`: 0대

카나리 인스턴스만 제거하고 기존 버전 100%로 복귀한다. 결과적으로 CANARY 진입 직전 상태와 같다.

### 3.4 deploy_force — 카나리 단계 생략

```yaml
canary:
  state: deploy_force
  version:
    normal: gateway-1.0.213
```

- `main`: 신버전 × `replicas(4)`로 즉시 전체 교체
- `sub`: 0대 *(직전 상태가 `finish`(정상 운영)일 때 기준)*

카나리 단계 없이 신버전을 전체에 즉시 배포한다. 트래픽 검증이 의미가 없는 변경(설정값, 로깅 레벨, 의존성 마이너 업데이트 등)이나 긴급 패치(취약점 대응, 장애 hotfix)에 사용한다.

위 결과는 직전 상태가 `finish`인 정상 경로를 가정한다. 직전 상태가 `canary`였다면 sub deployment의 카나리 인스턴스가 정리되지 않고 잔존하는 현상이 관찰되었다. 자세한 내용은 **5. 운영 주의점** 에서 다룬다.

---

## 4. 운영 흐름 예시

신기능을 카나리로 검증한 뒤 정식 배포하는 시나리오.

```
[월요일] PR #1  state=canary, version.canary=v1.0.213
  머지 → ArgoCD sync → sub 에 v1.0.213 인스턴스 1대 기동
  메트릭/로그 관찰 (수 시간 ~ 하루)

[화요일] PR #2  state=finish, version.normal=v1.0.213
  머지 → ArgoCD sync → main 4대가 v1.0.213으로 롤링 교체
  state=finish 정상 운영으로 복귀
```

검증 중 문제를 발견한 경우.

```
[월요일] PR #1  state=canary → 머지 → 카나리 1대 기동
  비즈니스 지표 이상 감지

[월요일] PR #2  state=rollback → 머지
  카나리 인스턴스 제거, 기존 버전 100% 운영으로 복귀
```

---

## 5. 운영 주의점: CANARY 직후 DEPLOY_FORCE

운영 중 다음 시퀀스에서 sub deployment의 카나리 인스턴스가 정리되지 않는 현상이 재현되었다.

```
PR #1  state=canary       → sub 에 카나리 1대 기동
PR #2  state=deploy_force → 곧바로 강제 배포
결과   sub 인스턴스가 제거되지 않고 잔존
```

반면 다음 두 경로에서는 sub deployment가 정상적으로 정리되는 것을 확인했다.

```
state=canary → state=finish    → sub 정리됨
state=canary → state=rollback  → sub 정리됨
```

sub deployment를 정리하는 책임은 실질적으로 `finish`와 `rollback` 두 상태에 있으며, `deploy_force`를 카나리 진행 중에 직접 거는 경로는 sub 정리 동작을 보장하지 않는다. 정확한 트리거가 chart template의 상태별 렌더링 차이인지 ArgoCD Application의 `prune` 설정 차이인지는 chart template 원문과 Application 정의를 함께 확인해야 단정할 수 있다.

### 안전한 진행 순서

운영상 검증된 우회 절차는 다음과 같다.

```
① state=rollback PR → 머지 → 카나리 인스턴스부터 정리
② state=deploy_force PR → 머지 → 강제 전체 배포
```

규칙은 **CANARY 상태에서 다른 상태로 직접 전이하지 않고, 반드시 finish 또는 rollback을 거쳐 sub deployment를 정리한 뒤 다음 상태로 전이한다** 이다. PR 템플릿이나 운영 가이드에 명시해두면 재발을 막을 수 있다.

---

## 6. 운영 시 고려할 점

- **카나리 트래픽 5–10%**. New Relic 가이드는 신버전 트래픽 비중을 5–10%로 권장한다. 비중이 너무 낮으면 엣지 케이스를 놓치고, 너무 높으면 영향 범위가 커진다.
- **finish 전 관찰 구간**. 비즈니스 트래픽 패턴에 따라 다르지만, 한 주기(피크-오프피크)를 포함하는 시간을 확보하는 편이 안전하다.
- **카나리 중 변경 격리**. 카나리가 진행되는 동안 무관한 변경을 함께 머지하면 이상 신호의 원인을 분리하기 어렵다.
- **canary 상태 방치 금지**. main과 sub이 서로 다른 버전으로 공존하는 시간이 길어질수록 디버깅 비용이 증가한다.