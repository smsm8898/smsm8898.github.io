---
title: "Autoscaling · Disruption"
date: 2026-07-28T23:10:00+09:00
tags: ["Kubernetes", "HPA", "PodDisruptionBudget", "replicas", "Metrics Server"]
categories: ["Kubernetes"]
summary: "몇 개로 실행하고 몇 개까지 죽어도 되는가 — HPA와 replicas가 충돌하는 이유, PDB가 노드 작업을 막는 방식, 오토스케일링이 무의미한 워크로드"
---

몇 개로 실행할지, 그리고 그중 몇 개까지 죽어도 되는지를 정하는 층. 이번 글은 HPA와 PDB 편.

## 1. 정의

- **HPA(HorizontalPodAutoscaler)**: 메트릭을 보고 replicas를 자동으로 조정한다.
  "CPU 사용률이 70%를 넘으면 늘려라"
- **PDB(PodDisruptionBudget)**: 자발적 중단(voluntary disruption) 시 **동시에 죽일 수 있는
  Pod 수의 하한**을 정한다. "최소 1개는 항상 살아있어야 한다"
- **두 종류의 중단을 구분해야 한다**

| 구분 | 예 | PDB 적용 |
|---|---|---|
|자발적(voluntary)|노드 drain, 클러스터 업그레이드, 노드 통합|**적용됨** (PDB가 막는다)|
|비자발적(involuntary)|노드 하드웨어 장애, OOMKilled, 커널 패닉|적용 안 됨 (막을 수 없다)|

  - PDB는 "죽지 않게" 만드는 장치가 아니라 **"한꺼번에 죽지 않게"** 만드는 장치다

- **스케일링의 세 축**

|축|리소스|무엇을 늘리나|
|---|---|---|
|수평(horizontal)|HPA|Pod 개수|
|수직(vertical)|VPA|Pod 하나의 requests·limits|
|노드|Cluster Autoscaler / Karpenter|노드 개수|

  - HPA가 Pod을 늘려도 노드에 자리가 없으면 Pending이다. 즉 HPA는 노드 오토스케일러와
    짝일 때 완성된다

## 2. 종류

### 2-1. replicas — 고정 개수

```yaml
spec:
  replicas: 2
```

- 가장 단순하다. 트래픽 변동이 없거나, 앱이 복제를 지원하지 않으면 이게 정답
- **HPA를 함께 쓸 때는 이 필드를 아예 두지 않는다** (참고 1번)

### 2-2. HPA — 메트릭 기반 조정

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:                 # 무엇을 조정할까
    apiVersion: apps/v1
    kind: Deployment
    name: product-personalize-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # requests 대비 비율
```

- **`averageUtilization`은 requests 대비 비율이다**: 노드 용량이나 limits가 아니다.
  requests가 100m인데 실제로 70m을 쓰면 70%. 그래서 **requests를 잘못 잡으면 HPA도 잘못 동작한다**
- **Metrics Server가 필요하다**: `type: Resource` 메트릭은 metrics-server가 제공한다.
  없으면 HPA가 `unknown` 상태로 아무것도 하지 않는다
  - Prometheus 메트릭(초당 요청 수, 큐 길이)으로 스케일하려면 Prometheus Adapter나
    KEDA 같은 추가 컴포넌트가 필요하다
- **`minReplicas`가 가용성의 하한이다**: HPA가 붙으면 replicas는 이 값 아래로 내려가지 않는다.
  단일 장애점을 피하려면 2 이상

### 2-3. PDB — 동시 중단 제한

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 1        # 또는 maxUnavailable: 1
  selector:
    matchLabels:
      app-name: product-personalize-api
```

- `minAvailable`과 `maxUnavailable` 중 **하나만** 쓸 수 있다. 절대값(1)이나 비율(`50%`) 모두 가능
- **동작 방식**: `kubectl drain`이나 노드 통합이 Pod을 축출하려 할 때, PDB를 위반하면
  **축출 요청이 거부된다**(429). 축출하는 쪽이 재시도하며 기다린다
- **replicas 1 + minAvailable 1은 노드 작업을 영구히 막는다** (참고 2번)

## 3. 대안

- **개수를 정하는 방법**

|방법|어울리는 곳|주의|
|---|---|---|
|`replicas` 고정|트래픽이 일정한 내부 API, 상태 있는 앱|피크에 수동 대응|
|HPA (CPU)|CPU 바운드 워크로드|I/O 바운드는 CPU가 안 올라 스케일이 안 됨|
|HPA (커스텀 메트릭)|큐 길이·RPS 기반|Adapter/KEDA 추가 필요|
|스케일 안 함|배치 job, DaemonSet|개수 개념이 다름|

- **상태 있는 앱은 복제가 답이 아니다**

|앱|왜 replicas 1인가|HA 방법|
|---|---|---|
|Grafana (기본 SQLite)|여러 Pod이 같은 파일을 쓰면 깨짐|외부 DB(Postgres)로 교체 후 복제|
|Airflow scheduler|복제 가능하지만 DB 부하·락 경합 증가|먼저 수직 확장, 필요시 2개|
|Prometheus|각 Pod이 따로 스크랩해 데이터가 갈림|Thanos·Mimir로 분리|

  - **"replicas를 늘리면 가용성이 올라간다"는 상태 없는 앱에서만 참이다.** 상태 있는 앱은
    스토리지·정합성 설계를 먼저 바꿔야 한다

- **PDB 값을 어떻게 잡을까**

|replicas|권장|이유|
|---|---|---|
|1|PDB를 두지 않음|`minAvailable: 1`이면 노드 작업이 영구 차단|
|2|`minAvailable: 1`|하나씩 교체 가능|
|3+|`maxUnavailable: 1` 또는 `minAvailable: 50%`|여러 개를 동시에 빼도 되게|

## 4. 참고

**1. HPA와 replicas를 함께 쓰면 서로 되돌린다**

Deployment에 `replicas: 2`가 적혀 있고 HPA가 10으로 올렸다면, 다음 sync에서
GitOps 도구가 "선언된 값은 2인데 실제가 10이네"라고 판단해 2로 되돌린다. HPA는 다시 10으로
올리고, 이 싸움이 반복된다.

- 해결은 **HPA를 쓸 때 Deployment에서 `replicas`를 아예 렌더하지 않는 것**이다

  ```yaml
  spec:
    {{- if not $cfg.hpa.enabled }}
    replicas: {{ $cfg.replicas }}
    {{- end }}
  ```

- 필드가 없으면 기본값 1로 생성되지만, HPA가 곧 `minReplicas`까지 올린다. 초기 순간에만
  1개인 것이 신경 쓰이면 ArgoCD의 `ignoreDifferences`로 replicas 차이를 무시하는 방법도 있다

**2. replicas 1에 PDB를 걸면 노드를 비울 수 없다**

`minAvailable: 1`에 Pod이 1개면, 그 Pod을 축출하는 순간 조건이 깨진다. 그래서 축출이
영구히 거부되고 `kubectl drain`이 끝나지 않는다.

- 노드 통합·업그레이드가 이 Pod 하나 때문에 멈춘다. 클러스터 오토스케일러도 그 노드를
  줄이지 못한다
- 그래서 PDB는 replicas 2 이상일 때 의미가 있다. replicas 1이면 PDB를 두지 않거나,
  `maxUnavailable: 1`로 두어 축출을 허용한다

**3. HPA가 아무 반응이 없을 때 확인 순서**

```text
1. metrics-server 가 설치돼 있나        → 없으면 메트릭이 unknown
2. Pod 에 requests 가 있나              → 없으면 비율 계산 자체가 불가
3. scaleTargetRef 이름이 맞나           → Deployment 이름 오타
4. 실제 사용률이 target 을 넘나          → I/O 바운드면 CPU 가 안 오른다
```

- 특히 2번이 놓치기 쉽다. `averageUtilization`은 requests 대비 비율이므로 requests가 없으면
  HPA는 계산할 기준이 없다. 스케줄링 편에서 "리소스를 안 적는 것이 위험하다"고 한 이유가 하나 더 늘어난다
- I/O 바운드 워크로드(외부 API 대기, DB 쿼리 대기)는 부하가 올라도 CPU가 안 오른다.
  이 경우 CPU 기반 HPA는 동작하지 않으므로 큐 길이나 RPS 기반으로 바꿔야 한다

**4. 오토스케일링과 알림의 관계 — max 도달은 정상인가**

HPA가 `maxReplicas`에 도달했다는 알림(`KubeHpaMaxedOut`)은 노이즈가 되기 쉽다.
피크마다 max에 닿는 것은 오토스케일러가 **정상 동작하고 있다는 뜻**이기 때문이다.

- 취할 액션이 없는 알림은 끄는 것이 맞다. 대신 진짜 문제(스케일아웃으로도 부족해 포화된 상태)는
  컨테이너 리소스 근접 알림(`ContainerCPUNearLimit`)이 잡는다
- 알림을 설계할 때 기준은 "이 알림을 받으면 무엇을 할 것인가"다. 답이 없으면 그 알림은 룰만
  남기고 라우팅에서 무음 처리한다

**5. 노드 오토스케일링이 워크로드 알림을 오탐으로 만든다**

노드가 자주 추가·제거되는 환경에서는 per-node 타겟(kubelet, node-exporter)이
scale-in마다 down으로 잡힌다. 비율 기반 `TargetDown` 알림은 이것을 장애로 오인한다.

- 대응: per-node job은 비율 계산에서 제외하고, 대신 **전멸 감시**로 바꾼다

  ```text
  # 비율 기반에서 제외
  up{job!~"kubelet|node-exporter"}

  # 전멸만 감시
  absent(up{job="node-exporter"} == 1)
  ```

- 노드가 몇 개 빠지는 것은 정상이고, **전부 없어지는 것**은 비정상이라는 구분

**6. 스케일 정책을 손대야 하는 경우**

기본 동작은 급격한 증가에는 빠르게, 감소에는 느리게(안정화 5분) 반응한다.
대부분 적절하지만, 트래픽이 스파이크성이면 축소가 너무 느려 비용이 든다.

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300    # 기본값. 짧게 하면 flapping 위험
```

- 줄이기 전에 확인할 것: 축소가 느린 것이 문제인가, 아니면 애초에 `minReplicas`가 높은가.
  후자가 원인인 경우가 더 많다
