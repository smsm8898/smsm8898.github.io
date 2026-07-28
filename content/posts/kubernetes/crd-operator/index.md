---
title: "CRD · Operator"
date: 2026-07-28T23:15:00+09:00
tags: ["Kubernetes", "CRD", "Operator", "ServiceMonitor", "PodMonitor", "PrometheusRule", "ServerSideApply"]
categories: ["Kubernetes"]
summary: "리소스 종류를 직접 추가하는 법 — CRD와 Operator가 만드는 위임 구조, 라벨 셀렉터가 조용히 끊어지는 사고, 거대 CRD가 apply를 실패시키는 문제"
---

지금까지는 Kubernetes가 제공하는 리소스를 썼다. 이번 글은 **리소스 종류를 직접 만들어 쓰는** 편.

## 1. 정의

- **CRD(CustomResourceDefinition)**: Kubernetes API에 **새로운 리소스 종류를 등록**하는 리소스.
  등록하면 그때부터 `kubectl get prometheus` 같은 명령이 동작한다
- **CR(Custom Resource)**: 그 종류로 만든 실제 오브젝트. CRD가 `Deployment`라는 개념이라면
  CR은 `product-personalize-api`라는 인스턴스
- **CRD만으로는 아무 일도 일어나지 않는다**: CRD는 "이런 필드를 가진 오브젝트를 저장할 수 있다"는
  스키마 등록일 뿐이다. etcd에 저장되고 조회될 뿐, 파드가 뜨지는 않는다
- **Operator**: 그 CR을 watch하다가 실제 리소스를 만드는 컨트롤러. **CRD + Operator**가
  한 쌍이어야 동작한다

  ```text
  CRD          : Prometheus 라는 종류를 등록
  CR           : "replicas 1, retention 7d 인 Prometheus 를 원한다"
  Operator     : 그 CR 을 읽어 StatefulSet · ConfigMap · Service 를 만든다
  ```

- **왜 이 구조를 쓰나**: 앱 고유의 운영 지식(스크랩 설정 생성, 롤링 재시작 순서, 샤딩)을
  Operator 코드에 담아두면, 사용자는 의도만 선언하면 된다. Deployment로는 표현할 수 없는 것들

## 2. 종류

### 2-1. Prometheus CR — 워크로드를 위임한다

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: monitoring-prometheus
spec:
  replicas: 1
  retention: 7d
  retentionSize: 8GB
  storage:
    emptyDir: { sizeLimit: 10Gi }
  serviceMonitorSelector: {}          # 어떤 ServiceMonitor 를 볼까
  podMonitorSelector: {}
  ruleSelector: {}
  externalLabels:
    cluster: dev
```

- 이 CR 하나가 StatefulSet·Service·ConfigMap(스크랩 설정)·Secret을 만들어낸다.
  **그래서 `helm template` 결과에는 StatefulSet이 없다** (참고 1번)
- `replicas`·`retention`을 바꾸면 Operator가 StatefulSet을 조정한다. 즉 CR이 소스이고
  StatefulSet은 파생물이라, StatefulSet을 직접 수정하면 되돌려진다
- **`*Selector` 필드가 이 리소스의 핵심**이다. Prometheus는 스크랩 대상을 자기 설정 파일에
  적어두지 않고, **클러스터에서 ServiceMonitor·PodMonitor를 찾아서** 설정을 조립한다

### 2-2. ServiceMonitor · PodMonitor — 스크랩 대상 선언

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: product-personalize-api
  labels:
    release: kube-prometheus-stack      # ← Prometheus 가 나를 찾는 라벨
spec:
  selector:
    matchLabels:
      app-name: product-personalize-api  # ← 내가 스크랩할 Pod 을 찾는 라벨
  podMetricsEndpoints:
    - port: deploy-port
      path: /metrics
      interval: 30s
```

- **라벨이 두 층으로 쓰인다**는 점이 처음에 혼란스럽다

  ```text
  Prometheus CR  --(podMonitorSelector)-->  PodMonitor  --(spec.selector)-->  Pod
       "어떤 모니터를 볼까"                        "어떤 파드를 긁을까"
  ```

- ServiceMonitor는 Service를 거쳐 엔드포인트를 찾고, PodMonitor는 Pod을 직접 찾는다.
  Service를 만들 이유가 없는 워커에는 PodMonitor가 맞다
- **네임스페이스 경계**: 기본적으로 Operator는 자기 네임스페이스만 watch한다.
  다른 네임스페이스의 모니터를 잡으려면 Operator 설정에서 그 네임스페이스를 열어야 한다 (참고 3번)

### 2-3. PrometheusRule — 알림·집계 규칙

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: reco-api.alerts
      rules:
        - alert: RecoHigh5xxErrorRate
          expr: |-
            sum by (namespace, job) (rate(http_requests_total{status="5xx"}[5m]))
              / sum by (namespace, job) (rate(http_requests_total[5m])) > 0.05
          for: 5m
          labels: { severity: critical }
          annotations:
            summary: "5xx 에러율 초과"
```

- 두 종류의 규칙이 같은 리소스에 들어간다
  - **alerting rule**(`alert:`): 조건을 만족하면 알림 발생
  - **recording rule**(`record:`): 무거운 쿼리를 미리 계산해 새 메트릭으로 저장
- **recording rule을 끄면 대시보드가 빈다**: 기본 대시보드들은 원시 메트릭이 아니라
  사전 계산된 메트릭을 조회한다. 알림이 시끄러워서 룰 그룹을 전부 끄면 대시보드가 함께 죽는다

### 2-4. CRD는 Prometheus만의 것이 아니다

지금까지 나온 것 중에도 CRD가 있었다.

| CRD | 제공자 | 쓰임 |
|---|---|---|
|`SecretProviderClass`|Secrets Store CSI Driver|외부 저장소 → k8s Secret 동기화|
|`Prometheus`, `Alertmanager`|prometheus-operator|모니터링 워크로드 선언|
|`ServiceMonitor`, `PodMonitor`, `PrometheusRule`, `Probe`|prometheus-operator|스크랩 대상·규칙|
|`Application`|ArgoCD|배포 단위 선언|
|`Ingress`|(내장 리소스)|CRD가 아니지만 컨트롤러 패턴은 동일|

- Ingress가 좋은 비교다. 리소스 자체는 내장이지만 **규칙만 선언하고 실제 로드밸런서는
  컨트롤러가 만든다**는 구조는 Operator와 같다. CRD·Operator는 이 패턴을 사용자가 확장할 수 있게 한 것

## 3. 대안

- **직접 만들까, Operator에 맡길까**

|방식|장점|단점|어울리는 곳|
|---|---|---|---|
|StatefulSet 직접 작성|투명함 — 렌더 결과가 최종 상태|스크랩 설정·재시작 순서를 직접 관리|단순한 앱|
|CR + Operator|운영 지식이 내장, 선언이 간결|렌더로 최종 상태를 알 수 없음, Operator 자체가 장애점|Prometheus·Kafka 같은 복잡한 앱|

- **스크랩 대상을 선언하는 방법**

|방법|Service 필요|어울리는 곳|
|---|---|---|
|ServiceMonitor|필요|이미 Service가 있는 컴포넌트|
|PodMonitor|불필요|워커·사이드카, Service를 만들 이유가 없는 것|
|`scrape_configs` 직접|—|Operator 밖의 대상(외부 exporter)|

- **CRD 배포 방식**

|방식|업그레이드|위험|
|---|---|---|
|차트에 포함(`crds/`)|Helm이 **업그레이드하지 않음**(설계상)|CRD 스키마 변경이 반영 안 됨|
|별도 apply|수동 관리|버전 불일치|
|Operator가 관리|자동|Operator 권한이 커짐|

  - Helm은 `crds/` 디렉토리의 CRD를 **설치 시에만** 만들고 업그레이드하지 않는다.
    차트 버전을 올릴 때 CRD를 따로 apply해야 하는 경우가 생기는 이유

## 4. 참고

**1. `helm template` 결과에 최종 워크로드가 없다**

kube-prometheus-stack을 렌더하면 워크로드가 이렇게 나온다.

```text
Deployment: kube-prometheus-stack-grafana
Deployment: kube-prometheus-stack-kube-state-metrics
Deployment: monitoring-operator
DaemonSet:  kube-prometheus-stack-prometheus-node-exporter
```

Prometheus와 Alertmanager의 StatefulSet이 없다. 차트가 만드는 것은 CR이고, StatefulSet은
Operator가 런타임에 만들기 때문이다.

- **CR을 쓰는 차트를 검증할 때는 두 층을 봐야 한다** — 렌더된 CR의 필드가 맞는지, 그리고
  그 CR이 어떤 리소스로 펼쳐지는지
- 이 구조의 이점: `retention`을 바꾸면 Operator가 StatefulSet을 알아서 조정한다.
  직접 StatefulSet을 썼다면 스토리지·설정·재시작을 모두 손으로 맞춰야 한다

**2. 셀렉터가 라벨 매칭이라 조용히 끊어진다**

가장 값비싼 사고였다. Prometheus CR의 기본 셀렉터는 **라벨 매칭**이다.

```yaml
serviceMonitorSelector:
  matchLabels:
    release: "kube-prometheus-stack"
```

wrapper 차트에 `_helpers.tpl`을 추가하면서 upstream과 **같은 이름**의 라벨 헬퍼를 정의했더니,
Helm의 named template이 전역이라 부모 차트 것이 subchart 것을 덮어썼다. 그 결과 upstream이
만드는 리소스에서 `release` 라벨이 사라졌다.

```text
release 라벨 개수 :  120개  →  15개
```

- ServiceMonitor·PrometheusRule이 전부 셀렉터에서 빠졌고, **대시보드가 No data**가 됐다.
  에러는 아무 곳에도 없다 — 스크랩 대상이 0개인 것은 정상 상태와 구분되지 않는다
- 해결은 셀렉터를 "전체 선택"으로 바꾸는 것

  ```yaml
  ruleSelectorNilUsesHelmValues: false
  serviceMonitorSelectorNilUsesHelmValues: false
  podMonitorSelectorNilUsesHelmValues: false
  ```

  `{}`(빈 셀렉터)를 지정하는 것만으로는 안 된다. Helm 템플릿에서 `{}`는 falsy라서
  "지정하지 않음"으로 취급되고, 그때 기본값인 라벨 매칭으로 떨어진다

- **더 나은 예방책은 헬퍼 이름을 겹치지 않게 짓는 것**이다. `kube-prometheus-stack.labels`가
  아니라 `my-wrapper.labels`로 정의했다면 충돌 자체가 없었다

**3. 부분적으로만 고치면 남은 곳에서 같은 사고가 재발한다**

위 문제를 `rule`·`serviceMonitor`·`podMonitor` 세 개만 풀었다. 렌더를 보면 나머지가 남아있다.

```yaml
probeSelector:
  matchLabels:
    release: "kube-prometheus-stack"      # ← 여전히 라벨 매칭
```

- 나중에 blackbox exporter로 외부 엔드포인트를 감시하려고 `Probe` CR을 만들면, 라벨을
  붙이지 않는 한 조용히 무시된다
- 교훈: 같은 종류의 필드가 여러 개 있으면 **전부 확인**해야 한다. 셋만 고치고 "해결됐다"고
  기록하면 남은 하나가 몇 달 뒤에 터진다

**4. CRD가 커서 apply가 실패한다**

prometheus-operator의 CRD는 하나가 256KB를 넘는다. client-side apply는 이전 상태를
`last-applied-configuration` **annotation에 저장**하는데, annotation 총 크기 제한이 256KB라
적용 자체가 실패한다.

```text
metadata.annotations: Too long: must have at most 262144 bytes
```

- ArgoCD에서는 sync마다 재발해 OutOfSync가 고착된다. 대응은 server-side apply로 바꾸는 것

  ```yaml
  syncOptions:
    - ServerSideApply=true
    - ApplyOutOfSyncOnly=true      # 이미 동기화된 리소스는 건너뛴다
  ```

- server-side apply는 이전 상태를 annotation이 아니라 `metadata.managedFields`에 기록하므로
  이 제한에 걸리지 않는다

**5. CRD를 지우면 그 종류의 모든 오브젝트가 함께 지워진다**

CRD 삭제는 그 CRD로 만든 **모든 CR을 cascade 삭제**한다. `Prometheus` CRD를 지우면
Prometheus CR이 사라지고, Operator가 그것을 보고 StatefulSet까지 정리한다.

- 그래서 차트를 uninstall할 때 CRD를 함께 지우지 않는 것이 기본 동작이다.
  Helm은 `crds/`의 CRD를 삭제하지 않는다
- ArgoCD에서 `prune: true`를 쓸 때 CRD가 prune 대상에 들어가지 않는지 확인해야 한다.
  라벨 하나 잘못 건드려 CRD가 관리 대상에서 빠지면, 다음 sync에서 지워질 수 있다

**6. CR을 만들었는데 아무 일도 일어나지 않는다**

CRD·Operator가 다 있는데도 CR이 무시되는 경우가 있다. 확인 순서는 이렇다.

```text
1. CRD 가 등록돼 있나        → kubectl get crd | grep monitoring
2. Operator 가 떠 있나       → 죽어 있으면 CR 은 그냥 etcd 에 앉아만 있다
3. Operator 가 그 네임스페이스를 watch 하나
4. CR 이 셀렉터에 걸리나      → 참고 2번
```

- 3번이 놓치기 쉽다. Operator는 기본적으로 자기 네임스페이스만 본다

  ```yaml
  prometheusOperator:
    namespaces:
      releaseNamespace: true
      additional:
        - reco          # 이 네임스페이스의 PodMonitor 도 보게 한다
  ```

- 렌더에서 Operator의 실행 인자로 확인할 수 있다: `--namespaces=<릴리스ns>,reco`
- **Operator가 단일 장애점이다**: Operator가 죽으면 새 CR이 반영되지 않는다. 이미 만들어진
  StatefulSet은 계속 돌지만, 설정 변경·스크랩 대상 추가가 멈춘다. 그래서 Operator 자체의
  건강도 감시 대상이다
