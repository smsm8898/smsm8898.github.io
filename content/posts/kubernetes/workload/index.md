---
title: "Workload"
date: 2026-07-28T21:00:00+09:00
tags: ["Kubernetes", "Pod", "Deployment", "StatefulSet", "DaemonSet", "Job", "CronJob"]
categories: ["Kubernetes"]
summary: "무엇을 어떻게 실행할지 정하는 리소스 6종 — Pod와 컨트롤러들의 차이, 언제 무엇을 쓰나, 그리고 직접 만들지 않고 Operator·pod template에 위임하는 두 가지 방식까지"
---

리소스 계층의 첫 층. 이번 글은 **무엇을 실행하는가**를 정하는 워크로드 편.

## 1. 정의

- **Pod**: 컨테이너 하나 이상을 묶은 실행 단위. 같은 Pod 안의 컨테이너는 네트워크(localhost)와
  볼륨을 공유한다. Kubernetes가 스케줄링하는 최소 단위가 컨테이너가 아니라 Pod인 이유
- **Pod은 일회용이다**: 노드가 죽거나 축출되면 그 Pod은 되살아나지 않는다. 이름도 IP도 그대로
  복구되지 않음 → 그래서 Pod을 직접 만들지 않고 **컨트롤러**에게 "이런 Pod을 n개 유지해줘"라고 맡긴다
- **컨트롤러의 공통 구조**: 어느 컨트롤러든 `spec.template`에 Pod 명세를 품고 있고, 그 위에
  자기만의 규칙(몇 개, 어떤 순서, 언제)을 얹는다

  ```yaml
  spec:
    replicas: 2          # 컨트롤러의 규칙
    selector:            # 내가 관리할 Pod을 찾는 라벨 조건
      matchLabels:
        app-name: product-personalize-api
    template:            # ← 이 아래는 그냥 Pod 명세
      metadata:
        labels:
          app-name: product-personalize-api   # selector 와 반드시 일치
      spec:
        containers: [...]
  ```

  - `selector`와 `template.metadata.labels`가 어긋나면 컨트롤러가 자기가 만든 Pod을 못 찾는다.
    Deployment는 이 경우 생성 자체를 거부한다

## 2. 종류

### 2-1. Deployment — 상태 없는 앱의 기본값

교체 가능한 Pod을 n개 유지한다. Pod들이 서로 구분되지 않아도 되는 모든 경우에 쓴다.

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 정원보다 몇 개 더 띄울 수 있나
      maxUnavailable: 0    # 몇 개까지 없어도 되나
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: product-personalize-api
          livenessProbe:                 # 죽었나? → 실패하면 컨테이너 재시작
            httpGet: { path: /health, port: deploy-port }
            failureThreshold: 3
          readinessProbe:                # 받을 준비됐나? → 실패하면 Service 에서 제외
            httpGet: { path: /health, port: deploy-port }
          lifecycle:
            preStop:
              exec: { command: ["/bin/sh", "-c", "sleep 30"] }
```

- **`maxUnavailable: 0`은 무중단 배포의 조건**: 새 Pod이 Ready가 된 뒤에야 구 Pod을 죽인다.
  대신 교체 순간 정원+1개가 동시에 존재하므로 **그만큼의 여유 리소스가 노드에 있어야 한다**
  (없으면 신규 Pod이 Pending → 참고 6번)
- **liveness와 readiness는 목적이 다르다**
  - liveness 실패 → 컨테이너 **재시작**. 그래서 `failureThreshold`를 넉넉히 줘야 한다. 바쁜 Pod이
    응답을 늦게 준 것을 죽은 것으로 오판하면 재시작 폭풍이 난다
  - readiness 실패 → Service의 엔드포인트에서 **제외**(재시작 없음). 부팅 중이거나 일시적으로
    과부하일 때 트래픽만 빼는 장치
- **`preStop` + `terminationGracePeriodSeconds`는 짝이다**: Pod 종료는 "엔드포인트 제거"와
  "SIGTERM 전달"이 **동시에** 일어난다. 로드밸런서(ALB) 쪽 디레지스터가 끝나기 전에 앱이 죽으면
  그 사이 들어온 요청이 504가 된다 → `preStop`으로 30초 버티게 하고, grace period를
  그보다 크게(60초) 줘야 sleep 중에 SIGKILL을 맞지 않는다

### 2-2. StatefulSet — 정체성이 필요한 앱

Pod마다 고정된 이름·순서·저장소를 준다. Pod이 죽고 다시 떠도 `<name>-0`은 여전히 `<name>-0`이고,
같은 PVC에 다시 붙는다.

```yaml
apiVersion: apps/v1
kind: StatefulSet
spec:
  serviceName: airflow-triggerer   # headless Service 이름 (DNS 를 위해)
  replicas: 1
  volumeClaimTemplates:            # Pod 마다 PVC 를 하나씩 만들어 준다
    - metadata: { name: logs }
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: 5Gi } }
```

- **Deployment와의 결정적 차이 세 가지**
  - 이름이 고정된다(`-0`, `-1`) → 각 Pod에 DNS 이름이 생김(`airflow-triggerer-0.airflow-triggerer`)
  - 생성·삭제가 순서대로 진행된다(0 → 1 → 2, 삭제는 역순)
  - `volumeClaimTemplates`로 Pod별 PVC를 자동 생성하고, 재생성 시 **같은 PVC에 다시 붙는다**
- **`serviceName`이 필수인 이유**: 각 Pod의 DNS 이름은 headless Service를 통해 부여된다.
  네트워크 편에서 다룰 내용이지만, StatefulSet 혼자로는 개별 Pod 주소가 생기지 않는다
- **PVC가 붙으면 스케줄링이 제약된다**: EBS 같은 RWO 볼륨은 특정 AZ에 묶여 있어, Pod이
  다른 AZ 노드로 갈 수 없다 → 노드가 축출되면 재스케줄이 실패할 수 있다 (참고 2번)

### 2-3. DaemonSet — 모든 노드에 하나씩

노드 단위로 정확히 하나의 Pod을 띄운다. 노드가 추가되면 자동으로 따라 뜬다.

```yaml
apiVersion: apps/v1
kind: DaemonSet
spec:
  template:
    spec:
      tolerations:
        - operator: Exists     # 모든 taint 를 허용 (핵심)
```

- **`replicas`가 없다**: 개수는 노드 수가 결정한다
- 노드 자체를 관측·중계하는 것들이 여기 해당한다 — 노드 메트릭 수집기(node-exporter),
  로그 수집기, CNI, 스토리지 드라이버
- **`tolerations: [operator: Exists]`가 사실상 필수**: taint가 걸린 노드를 허용하지 않으면
  그 노드에는 Pod이 안 뜨고, **그 노드의 메트릭·로그만 조용히 비어버린다** (참고 5번)

### 2-4. Job — 한 번 끝까지 실행

완료(exit 0)를 목표로 하는 작업. 성공하면 다시 실행하지 않는다.

```yaml
apiVersion: batch/v1
kind: Job
spec:
  backoffLimit: 0                 # 재시도 횟수 (0 = 재시도 안 함)
  activeDeadlineSeconds: 5400     # 이 시간을 넘기면 실패 처리
  ttlSecondsAfterFinished: 86400  # 끝난 Job 오브젝트를 언제 지울까
  template:
    spec:
      restartPolicy: Never        # Job 은 Always 를 쓸 수 없다
```

- `restartPolicy`는 `Never` 또는 `OnFailure`만 가능하다. `Always`는 "끝나지 않는 것"이라
  Job의 정의와 모순
- **`backoffLimit`을 0으로 두는 경우**: 재시도로 해결되지 않는 작업(예: 스키마 충돌, FK 위반).
  기본값 6은 실패를 6번 반복하며 로그만 불린다
- DB 마이그레이션, 초기 사용자 생성처럼 배포 시 한 번 도는 작업에 쓴다. 다만 GitOps와는
  상성이 나쁘다 (참고 3번)

### 2-5. CronJob — 주기적으로 Job 생성

스케줄에 맞춰 Job을 만들어내는 상위 컨트롤러.

```yaml
apiVersion: batch/v1
kind: CronJob
spec:
  schedule: "*/15 * * * *"
  concurrencyPolicy: Forbid       # 이전 실행이 안 끝났으면 건너뛴다
  startingDeadlineSeconds: 600
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec: { ... }                 # ← Job 명세 그대로
```

- **`concurrencyPolicy`가 중요한 이유**: 기본값 `Allow`는 이전 실행이 끝나지 않아도 새로 시작한다.
  대량 삭제 같은 작업이 겹치면 DB 락 경합이 난다 → `Forbid`로 직렬화
- `Replace`는 이전 것을 죽이고 새로 시작. 최신 상태만 중요한 작업(예: 캐시 갱신)에 어울린다
- **정리 작업의 정석**: 종료된 Pod 청소, 오래된 이력 삭제처럼 "쌓이는 것"을 주기적으로 비운다.
  단, 무거운 작업은 반드시 **전용 Pod**으로 돌려야 한다 (참고 4번)

### 2-6. 직접 만들지 않는 두 가지 방식

워크로드를 항상 우리가 YAML로 쓰는 것은 아니다.

- **Operator에 위임**: Prometheus는 `Prometheus`라는 커스텀 리소스(CR)만 선언하고, 실제
  StatefulSet은 Operator가 런타임에 만든다. 그래서 `helm template` 결과에 StatefulSet이 없다
  (참고 1번). CRD 편에서 자세히 다룬다
- **템플릿만 주고 런타임 생성**: Airflow의 KubernetesExecutor는 task마다 Pod을 새로 만드는데,
  그 Pod 명세는 ConfigMap에 `pod_template_file.yaml`로 들어있고 scheduler가 그것을 읽어
  API로 Pod을 생성한다. 컨트롤러 없이 Pod이 직접 만들어지는 경우

  ```text
  # airflow.cfg
  [kubernetes_executor]
  pod_template_file = /opt/airflow/pod_templates/pod_template_file.yaml
  delete_worker_pods = True
  ```

## 3. 대안

- **어떤 컨트롤러를 쓸까**

|질문|답|
|---|---|
|Pod들이 서로 구분되지 않아도 되나?|Deployment|
|Pod마다 고정 이름·저장소가 필요한가?|StatefulSet|
|모든 노드에 하나씩 필요한가?|DaemonSet|
|끝나는 작업인가?|Job|
|끝나는 작업을 주기적으로?|CronJob|
|앱이 스스로 워크로드를 관리해야 하나?|CRD + Operator|

- **replicas 2 이상이 항상 좋은가**

|구분|Deployment (상태 없음)|StatefulSet (상태 있음)|
|---|---|---|
|복제 의미|처리량·가용성 모두 증가|**앱이 클러스터링을 지원해야** 의미 있음|
|RWO PVC|보통 안 씀|Pod마다 별개 PVC라 가능|
|주의|없음|SQLite 백엔드처럼 단일 파일을 쓰는 앱은 replicas 1 고정 (동시 접근 충돌)|

  - Grafana를 replicas 1로 두는 이유가 이것이다. 기본 백엔드가 SQLite라 여러 Pod이 같은 PVC를
    공유하면 깨진다. HA가 필요하면 외부 DB로 바꾸는 것이 먼저

- **로그·데이터를 어디에 둘까**

|방식|장점|단점|어울리는 곳|
|---|---|---|---|
|`emptyDir`|스케줄링 제약 없음, 즉시 사용|Pod 삭제 시 유실|dev, 재생성 가능한 캐시|
|PVC (RWO)|영속|AZ에 묶여 재스케줄 제약|prod 메타데이터·시계열|
|외부 스토리지(S3)|Pod 수명과 무관, 보존기간 관리 용이|네트워크 의존, 자격증명 필요|사라지는 Pod의 로그|

  - Airflow의 task Pod은 끝나면 사라지므로 로그를 S3로 보낸다(`remote_logging`). 반면
    Prometheus 시계열은 dev에서 `emptyDir`, prod에서 PVC로 갈랐다 — dev는 유실을 허용하는 대신
    노드 축출 시 재스케줄 실패를 피하는 선택

## 4. 참고

**1. StatefulSet이 렌더 결과에 없는데 클러스터에는 있다?**

kube-prometheus-stack을 `helm template`으로 렌더하면 워크로드가 이렇게 나온다.

```text
Deployment: kube-prometheus-stack-grafana
Deployment: kube-prometheus-stack-kube-state-metrics
Deployment: monitoring-operator
DaemonSet:  kube-prometheus-stack-prometheus-node-exporter
```

Prometheus와 Alertmanager의 StatefulSet이 없다. 차트가 만드는 것은 `Prometheus`/`Alertmanager`
**커스텀 리소스**이고, StatefulSet은 Operator가 그 CR을 보고 런타임에 만들기 때문이다.

- 그래서 `helm template`만으로는 최종 워크로드를 전부 알 수 없다. CR을 쓰는 차트라면
  "무엇이 Operator에게 위임되는가"를 따로 확인해야 한다
- 반대로 이 구조 덕분에 replicas·retention·스토리지를 CR 필드로 선언하면 Operator가
  StatefulSet을 알아서 조정한다

**2. PVC가 붙은 StatefulSet은 노드 축출에서 교착될 수 있다**

RWO(ReadWriteOnce) EBS 볼륨은 특정 AZ에 존재한다. Pod이 축출되어 다른 AZ 노드로
스케줄되려 하면 볼륨을 붙일 수 없어 Pending에 빠진다. 노드 자동 확장·통합이 활발한
환경에서는 이것이 반복적인 OutOfSync를 만든다.

- dev에서는 `emptyDir`로 바꿔 이 문제를 회피했다 — 데이터 유실을 허용하는 대신 안정적으로 뜨는 쪽
- prod에서는 PVC를 쓰되, 보존 크기를 볼륨 크기보다 작게 잡아 디스크 full을 막는다
  (`retention: 30d`, `retentionSize: 45GB`, PVC 50Gi)

**3. Job은 GitOps와 상성이 나쁘다**

ArgoCD는 sync를 재시도한다. Job은 이름이 고정되어 있고 완료된 Job은 남아 있으므로,
재시도가 `already exists`로 실패하거나 의도치 않게 다시 실행된다.

- Airflow의 DB 마이그레이션 Job과 초기 사용자 생성 Job을 모두 끈 이유가 이것이다.
  마이그레이션은 버전 업그레이드 때만 필요하고, 사용자 생성은 최초 1회면 된다 → **수동 실행**으로 빼고
  차트에서는 비활성화

  ```yaml
  migrateDatabaseJob: { enabled: false }
  webserver:
    defaultUser: { enabled: false }
  ```

- 끄면 리소스가 실제로 사라지는지 렌더로 확인할 수 있다

  ```bash
  helm template airflow . -f values.yaml -f values-dev.yaml | grep -c "kind: Job"
  ```

**4. 무거운 작업을 컴포넌트 Pod 안에서 돌리면 그 컴포넌트가 죽는다**

메타데이터 DB 이력을 지우는 작업(`airflow db clean`)을 scheduler Pod 안에서 실행하면,
그 명령이 쓰는 메모리가 **scheduler 컨테이너의 cgroup에 잡힌다**. limit을 넘으면 작업이 아니라
scheduler가 OOMKilled 된다.

- 그래서 전용 CronJob으로 분리하고, 그 Pod에만 넉넉한 limit(4Gi)을 준다
- 같은 이유로 `activeDeadlineSeconds`를 둔다. 대량 삭제가 길어져 다음 스케줄과 겹치는 것을 막음

**5. 실패한 Pod을 즉시 지우면 알림이 사라진다**

KubernetesExecutor는 task Pod을 계속 만들어내므로 정리가 필요하다. 그런데 실패 Pod까지
즉시 지우면, OOMKilled 상태가 kube-state-metrics의 스크랩 주기(30초) 안에 관측되지 않아
`KubeContainerOOMKilled` 알림이 **조용히 유실된다**.

```yaml
[kubernetes_executor]
delete_worker_pods: 'True'
delete_worker_pods_on_failure: 'False'   # 실패 Pod 은 남긴다
```

- 성공 Pod은 즉시 정리하고, 실패 Pod은 남긴 뒤 cleanup CronJob(15분)이 나중에 치운다.
  관측 시간을 확보하면서 무한 누적도 막는 구조
- DaemonSet의 taint 허용을 빼먹는 것도 같은 종류의 사고다. Pod이 안 뜬 노드는 에러를 내지 않고
  **메트릭만 없다**. 값이 없는 것은 알림이 발화하지 않으므로, 이런 침묵은 `absent()` 같은
  "메트릭 소멸" 감시로 잡아야 한다

**6. RollingUpdate가 Pending으로 교착할 수 있다**

`maxUnavailable: 0`은 교체 순간 구 Pod과 신 Pod을 동시에 유지한다. 메모리 요청이 큰 컴포넌트
여러 개가 같은 노드그룹에서 한꺼번에 교체되면, 노드 용량이 구+신을 감당하지 못해
신규 Pod이 영구 Pending이 된다.

- replicas가 1이면 RollingUpdate도 어차피 무중단이 아니다 → `Recreate`로 바꾸면 구 Pod을
  먼저 종료하므로 순간 초과분(surge)이 아예 없다

  ```yaml
  scheduler:
    strategy: { type: Recreate }
  ```

- 이 판단의 기준은 "무중단이 실제로 성립하는가"다. replicas 2 이상이고 여유 용량이 있으면
  RollingUpdate가 맞고, replicas 1이거나 리소스가 빡빡하면 Recreate가 안전하다
