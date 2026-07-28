---
title: "Scheduling"
date: 2026-07-28T23:05:00+09:00
tags: ["Kubernetes", "nodeSelector", "Taint", "Toleration", "Affinity", "QoS", "Resources"]
categories: ["Kubernetes"]
summary: "Pod을 어느 노드에서 실행할지 정하는 방법 — nodeSelector만으로 부족한 이유(taint), requests와 limits가 만드는 QoS 등급이 축출 순서를 바꾸는 방식, 그리고 Pending의 원인들"
---

워크로드를 어디서 실행할지 정하는 층. 이번 글은 스케줄링 편.

## 1. 정의

- **스케줄러가 하는 일**: Pending 상태의 Pod을 보고 조건에 맞는 노드를 골라 배치한다.
  조건은 두 종류 — **리소스가 충분한가**(requests), **배치가 허용되는가**(selector·taint·affinity)
- **`requests`가 스케줄링의 기준이다**: 스케줄러는 `limits`를 보지 않는다. 노드의 남은 용량을
  계산할 때 그 노드에 있는 Pod들의 **requests 합계**만 뺀다
  - 그래서 requests를 실제보다 작게 적으면 노드가 과밀해지고, 크게 적으면 노드가 비어도 Pending이 난다
- **배치 조건의 세 가지 방향**

| 장치 | 방향 | 의미 |
|---|---|---|
|`nodeSelector` / `affinity`|Pod → 노드|"나는 이런 노드에 가고 싶다"|
|`taint`|노드 → Pod|"나는 아무나 받지 않는다"|
|`toleration`|Pod → taint|"그 조건을 견딜 수 있다"|

  - **둘은 짝이다**: taint가 걸린 노드에 가려면 nodeSelector로 지목하는 것만으로는 안 되고,
    toleration도 함께 있어야 한다 (참고 1번)

## 2. 종류

### 2-1. nodeSelector — 가장 단순한 지목

```yaml
spec:
  nodeSelector:
    node-group-name: dev-ng
```

- 노드에 붙은 라벨과 **정확히 일치**하는 노드에만 배치된다. AND 조건만 가능
- 노드그룹 단위로 워크로드를 가르는 대부분의 경우에 충분하다

### 2-2. taint · toleration — 노드가 거부하는 쪽

```yaml
# 노드 쪽 (인프라 코드에서 설정)
taints:
  - key: node-group-name
    value: dev-ng
    effect: NoSchedule

# Pod 쪽
tolerations:
  - key: node-group-name
    operator: Equal
    value: dev-ng
    effect: NoSchedule
```

- **`effect` 세 가지**

|effect|의미|
|---|---|
|`NoSchedule`|toleration 없으면 새로 배치되지 않음 (기존 Pod은 유지)|
|`PreferNoSchedule`|가능하면 피함 (강제 아님)|
|`NoExecute`|toleration 없는 기존 Pod까지 **축출**|

- **toleration은 허가일 뿐 유도가 아니다**: toleration만 있으면 "그 노드에 갈 수도 있다"는 뜻이고,
  다른 노드에 배치될 수도 있다. 특정 노드로 **보내려면** nodeSelector가 필요하다
- **모든 taint를 허용하는 특수 케이스**

  ```yaml
  tolerations:
    - operator: Exists      # key 를 지정하지 않으면 모든 taint 허용
  ```

  DaemonSet에 쓴다. 노드 메트릭 수집기는 어떤 taint가 걸린 노드에서도 떠야 하기 때문 (참고 2번)

### 2-3. affinity — 표현식으로 조건 걸기

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: eks.amazonaws.com/nodegroup
              operator: In
              values: [reco-ng, reco-ng-2]      # OR 조건이 가능
```

- nodeSelector로 안 되는 것을 한다 — `In`/`NotIn`/`Exists` 같은 연산자, 여러 값 중 하나(OR)
- **`required...` vs `preferred...`**: 앞은 만족하지 않으면 Pending, 뒤는 가중치를 줘서 우선할 뿐
- **`IgnoredDuringExecution`**: 실행 중에는 재평가하지 않는다. 이미 배치된 Pod은 조건이 깨져도
  쫓아내지 않음
- **podAntiAffinity**: 노드가 아니라 다른 Pod을 기준으로 삼는다. "같은 노드에 내 복제본을 두지 마라"로
  단일 노드 장애에 대비. replicas가 2 이상일 때만 의미가 있다

### 2-4. topologySpreadConstraints — 균등 분산

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
```

- AZ·노드 단위로 Pod을 고르게 퍼뜨린다. antiAffinity가 "붙지 마라"라면 이쪽은 "고르게 퍼져라"
- `whenUnsatisfiable`이 `DoNotSchedule`이면 균등 배치가 불가능할 때 Pending이 된다.
  가용성보다 배치 성공이 중요하면 `ScheduleAnyway`

### 2-5. requests · limits — QoS 등급을 만든다

두 값의 관계가 Pod의 **QoS 등급**을 결정하고, 등급이 노드 압박 시 축출 순서를 정한다.

```yaml
# Guaranteed — 가장 늦게 축출된다
resources:
  requests: { cpu: 250m, memory: 2Gi }
  limits:   { cpu: 1500m, memory: 2Gi }     # memory 가 requests 와 동일

# Burstable — 중간
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 1000m, memory: 2Gi }

# BestEffort — 가장 먼저 축출된다
resources: {}
```

| 등급 | 조건 | 축출 순서 |
|---|---|---|
|Guaranteed|모든 컨테이너의 requests == limits (cpu·memory 모두)|마지막|
|Burstable|requests가 있고 limits와 다름|중간|
|BestEffort|requests·limits 둘 다 없음|처음|

- **엄밀히는 cpu·memory 모두 같아야 Guaranteed**다. 다만 축출은 memory 압박으로 일어나므로
  실무에서는 memory만 맞추는 절충도 쓴다
- **cpu와 memory의 초과 처리가 다르다**

|자원|limit 초과 시|
|---|---|
|CPU|**throttling** (느려짐, 죽지 않음)|
|Memory|**OOMKilled** (즉시 종료)|

  - 그래서 memory limit은 신중하게, cpu limit은 상대적으로 여유 있게 잡는다

## 3. 대안

- **노드를 고르는 세 방법**

|방법|표현력|어울리는 곳|
|---|---|---|
|`nodeSelector`|정확히 일치(AND)만|노드그룹 단위 분리 — 대부분|
|`nodeAffinity`|In/NotIn/Exists, OR|여러 노드그룹 중 하나, 조건이 복잡할 때|
|`podAntiAffinity`|다른 Pod 기준|복제본을 다른 노드에 흩기 (replicas 2+)|

- **워크로드를 노드그룹으로 가르는 이유**

|노드그룹|올리는 것|왜|
|---|---|---|
|컨트롤플레인용|scheduler, api-server, operator, Prometheus|무거운 작업에 밀려 죽으면 안 되는 것들|
|워커용|task pod, 배치 job|노드를 꽉 채워 쓰고, 필요하면 축출돼도 되는 것들|

  - Airflow에서 이 분리가 중요하다. task Pod이 노드를 점유해도 scheduler가 영향받지 않게 하려면
    `workers.nodeSelector`를 컨트롤플레인과 다른 노드그룹으로 둔다

- **QoS를 어디까지 올릴까**

|구분|Guaranteed|Burstable|
|---|---|---|
|축출 저항|강함|중간|
|리소스 효율|낮음 (peak 기준으로 예약)|높음 (평시 기준 예약 + 필요시 burst)|
|어울리는 곳|prod 의 핵심 컴포넌트|dev, 트래픽 변동이 큰 API|

  - prod에서 scheduler·api-server의 memory를 requests == limits로 맞춘 이유가 이것이다.
    dev는 Burstable로 두어 노드를 촘촘히 쓴다

## 4. 참고

**1. nodeSelector만 넣었는데 Pod이 Pending에 머문다**

가장 흔한 실수다. 노드그룹에 taint가 걸려 있으면 nodeSelector로 지목해도 스케줄러가 거부한다.

```text
0/5 nodes are available: 5 node(s) had untolerated taint {node-group-name: dev-ng}
```

- **증상이 조용한 경우가 더 위험하다**: kube-state-metrics를 노드그룹에 핀할 때 toleration을
  빼먹으면 그 Pod이 Pending에 빠지고, `kube_*` 메트릭이 전부 사라진다. 그 메트릭에 의존하는
  알림들은 값이 없으니 발화하지 않는다 → **모니터링이 죽었는데 알림도 없는** 상태
- 그래서 nodeSelector와 toleration은 항상 같이 쓴다고 생각하는 편이 안전하다

**2. DaemonSet에서 taint 허용을 빼먹으면 특정 노드만 관측이 빈다**

노드 메트릭 수집기는 **모든** 노드에서 떠야 한다. taint가 걸린 노드를 허용하지 않으면
그 노드에는 Pod이 안 뜨고, 에러 없이 그 노드의 메트릭만 없다.

```yaml
prometheus-node-exporter:
  tolerations:
    - operator: Exists      # 모든 taint 허용
```

- 이런 침묵은 임계치 알림으로 잡히지 않는다. "메트릭이 아예 없는 상태"를 감시해야 한다

  ```text
  absent(up{job="node-exporter"} == 1)
  ```

**3. RollingUpdate가 Pending으로 교착하는 것도 스케줄링 문제다**

`maxUnavailable: 0`은 교체 순간 구 Pod과 신 Pod을 동시에 유지한다. requests가 큰 컴포넌트
여러 개가 같은 노드그룹에서 한꺼번에 교체되면 노드 용량이 구+신을 감당하지 못한다.

- 계산은 requests 기준이다. 예를 들어 memory requests 5Gi인 컴포넌트 4개가 동시에 교체되면
  순간적으로 40Gi가 필요하다
- 대응은 두 방향 — 노드 용량을 늘리거나, `Recreate`로 바꿔 surge를 없애거나. replicas 1이면
  후자가 합리적이다 (RollingUpdate여도 어차피 무중단이 아니므로)

**4. requests를 실제 사용량보다 낮게 잡으면 노드 전체가 불안정해진다**

스케줄러는 requests 합계로만 판단하므로, 실제 사용량이 requests보다 크면 노드가
overcommit 상태가 된다. 이때 메모리 압박이 오면 kubelet이 Pod을 축출하기 시작하고,
축출 순서는 QoS 등급이 정한다.

- BestEffort(리소스 미지정) Pod이 있으면 그것이 먼저 죽는다. 그래서 **리소스를 안 적는 것이
  가장 위험하다** — 아무 때나 죽어도 되는 Pod이 되는 셈
- 반대로 limits만 크게 잡고 requests를 작게 두면 "스케줄링은 쉽지만 실행은 불안정한" 조합이 된다

**5. PVC가 붙으면 스케줄링이 AZ에 묶인다**

EBS 같은 RWO 볼륨은 특정 AZ에 존재한다. 그 볼륨을 쓰는 Pod은 같은 AZ의 노드로만 갈 수 있다.

- 노드 자동 확장·통합이 활발한 환경에서 문제가 된다. Pod이 축출됐는데 그 AZ에 여유 노드가
  없으면 다른 AZ로 갈 수 없어 Pending에 빠진다
- 대응: dev에서는 `emptyDir`로 바꿔 제약을 없애고(유실 허용), prod에서는 PVC를 쓰되
  노드그룹을 AZ 단위로 넉넉히 유지한다

**6. cpu limit은 왜 여유 있게 잡는가**

CPU limit을 초과하면 컨테이너가 죽지 않고 **throttling**된다. 죽지 않으니 안전해 보이지만,
응답 지연이 늘어 liveness probe가 실패하고 결국 재시작으로 이어질 수 있다.

- 특히 probe 타임아웃이 짧으면 "바쁜 Pod을 죽은 Pod으로 오판"하는 사고가 난다.
  `failureThreshold`를 넉넉히 주는 것이 짝이 되는 대응
- 반대로 memory limit은 초과 즉시 OOMKilled이므로 실측 기반으로 정해야 한다. 관측 없이
  추측으로 정한 memory limit이 가장 흔한 장애 원인이다
