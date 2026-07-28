---
title: "Config · Secret"
date: 2026-07-28T22:30:00+09:00
tags: ["Kubernetes", "ConfigMap", "Secret", "CSI", "IRSA", "Pod Identity", "ServiceAccount"]
categories: ["Kubernetes"]
summary: "값과 자격증명을 Pod에 넣는 방법 — ConfigMap의 마운트 방식 세 가지, Secrets Manager에서 CSI를 거쳐 Secret이 되는 체인, IRSA와 Pod Identity의 차이"
---

컨테이너 이미지는 환경에 관계없이 같아야 하고, 달라지는 값은 밖에서 넣어야 한다.
이번 글은 설정과 자격증명 편.

## 1. 정의

- **ConfigMap**: 평문 설정을 담는 key-value 오브젝트. 파일 한 덩어리도 값으로 넣을 수 있다
- **Secret**: 구조는 ConfigMap과 같지만 값이 base64로 인코딩되고, RBAC·감사 로그에서
  별도로 취급된다. **base64는 암호화가 아니다** — etcd 암호화를 켜지 않으면 평문과 다를 바 없음
- **주입 방식은 두 가지뿐이다**: 환경변수로 넣거나, 볼륨으로 마운트하거나

| 방식 | 갱신 반영 | 어울리는 것 |
|---|---|---|
|`env` / `envFrom`|**안 됨** (Pod 재시작 필요)|짧은 값, 앱이 시작 시 한 번 읽는 것|
|볼륨 마운트|자동 반영 (kubelet이 주기적으로 갱신)|설정 파일, 인증서, 자주 바뀌는 것|

- **왜 Secret을 git에 두면 안 되나**: 매니페스트는 코드로 관리해야 하는데 값은 비밀이어야 한다는
  모순 → 외부 비밀 저장소(AWS Secrets Manager)에 두고 클러스터가 가져오게 한다. 그 통로가 CSI

## 2. 종류

### 2-1. ConfigMap — 세 가지 마운트 방식

같은 ConfigMap도 쓰는 방법에 따라 결과가 다르다.

```yaml
# (1) 환경변수로 전부
envFrom:
  - configMapRef: { name: airflow-config }

# (2) 디렉토리로 마운트 — key 하나당 파일 하나
volumeMounts:
  - name: config
    mountPath: /opt/airflow/config

# (3) 파일 하나만 지정 위치에 — subPath
volumeMounts:
  - name: config
    mountPath: /opt/airflow/airflow.cfg
    subPath: airflow.cfg
    readOnly: true
```

- **`subPath`가 필요한 이유**: (2)처럼 디렉토리로 마운트하면 그 경로에 원래 있던 파일이
  **전부 가려진다**. `/opt/airflow`에 통째로 마운트하면 이미지 안의 다른 파일이 사라짐 →
  파일 하나만 얹으려면 `subPath`
- **단, `subPath`는 자동 갱신이 안 된다**: 볼륨 마운트의 장점인 "ConfigMap 수정 시 자동 반영"이
  subPath에서는 동작하지 않는다. 설정을 바꾸면 Pod을 재시작해야 함
- **파일 한 덩어리를 값으로**: 대시보드 JSON처럼 큰 파일도 ConfigMap의 값이 될 수 있다

  ```yaml
  data:
    reco-api-monitoring.json: |-
  {{ .Files.Get "dashboards/reco-api-monitoring.json" | indent 4 }}
  ```

  Helm의 `.Files.Get`으로 차트 안의 파일을 읽어 넣는 패턴. 1MB 제한이 있으므로 큰 파일은 주의

### 2-2. 라벨로 발견되는 ConfigMap

ConfigMap을 마운트하지 않고, **다른 컴포넌트가 라벨로 찾아가는** 방식도 있다.

```yaml
metadata:
  name: grafana-dashboard-reco-api
  labels:
    grafana_dashboard: "1"        # ← Grafana sidecar 가 이 라벨을 감시
data:
  reco-api-monitoring.json: |- ...
```

- Grafana Pod의 sidecar 컨테이너가 클러스터 전체에서 이 라벨을 가진 ConfigMap을 watch하다가,
  발견하면 내용을 자기 파일시스템에 쓰고 Grafana가 로드한다
- 덕분에 대시보드를 추가할 때 Grafana를 재배포할 필요가 없다. 대신 **연결 고리가 4개**여서
  하나만 어긋나도 조용히 실패한다

  ```text
  ConfigMap(라벨) → sidecar(라벨 감시, 폴더에 파일 생성) → provider(그 폴더를 읽음) → Grafana
  ```

### 2-3. Secret — 자격증명

```yaml
# 개별 key 를 env 이름으로 지정
env:
  - name: SLACK_BOT_TOKEN
    valueFrom:
      secretKeyRef:
        name: slack-airflow-token
        key: token

# Secret 의 모든 key 를 그대로 env 이름으로
envFrom:
  - secretRef:
      name: reco-api-secret
```

- `secretKeyRef`는 이름을 바꿔 넣을 수 있고, `envFrom`은 key가 그대로 env 이름이 된다.
  후자는 Secret의 key를 앱이 기대하는 이름과 맞춰 만들어야 함
- **파일로 마운트해야 하는 경우**: 인증서, SSH 키처럼 파일 경로를 요구하는 것

  ```yaml
  volumeMounts:
    - name: git-secret
      mountPath: /etc/git-secret/ssh
      subPath: gitSshKey        # Secret 의 key 이름
      readOnly: true
  ```

### 2-4. SecretProviderClass — 외부 저장소에서 가져오기

Secrets Store CSI Driver가 제공하는 커스텀 리소스. 두 부분으로 나뉜다.

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: airflow-secrets-store
spec:
  provider: aws
  parameters:
    usePodIdentity: "true"
    objects: |                                    # (1) 무엇을 어떻게 꺼낼까
      - objectName: "dev/airflow/secrets"
        objectType: secretsmanager
        jmesPath:
          - path: pg_conn                          # JSON 안의 필드
            objectAlias: airflow-metadata-secret   # 꺼낸 값의 별칭
  secretObjects:                                   # (2) 어떤 k8s Secret 을 만들까
    - secretName: airflow-metadata-secret
      type: Opaque
      data:
        - key: connection                          # Secret 의 key
          objectName: airflow-metadata-secret      # 위 별칭
```

- **`parameters.objects`**: 외부 저장소의 어떤 secret을 읽고, JSON 안에서 어떤 필드를(`jmesPath`)
  어떤 별칭으로 꺼낼지. 하나의 JSON secret에서 여러 값을 뽑아낼 수 있다
- **`secretObjects`**: 그 별칭들을 조합해 만들 k8s Secret. 이게 있어야 `secretKeyRef`로 참조 가능
- **⚠️ Pod이 볼륨을 마운트해야 Secret이 만들어진다**: `secretObjects`는 선언만으로 실행되지 않는다.
  CSI 볼륨이 실제로 마운트되는 순간 동기화가 발동한다 (참고 1번)

  ```yaml
  volumes:
    - name: secrets-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: airflow-secrets-store
  volumeMounts:
    - name: secrets-store
      mountPath: /mnt/secrets-store      # 이 마운트가 트리거
      readOnly: true
  ```

### 2-5. ServiceAccount — Pod의 신분

Pod이 Kubernetes API와 AWS API에 자기를 증명하는 수단.

```yaml
# 방식 A: IRSA — SA 에 IAM role 을 annotation 으로 붙인다
apiVersion: v1
kind: ServiceAccount
metadata:
  name: airflow-service-account
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::<ACCOUNT_ID>:role/dev-airflow-sa-role"
    eks.amazonaws.com/audience: "sts.amazonaws.com"
```

```yaml
# 방식 B: Pod Identity — annotation 이 없다
apiVersion: v1
kind: ServiceAccount
metadata:
  name: reco-sa
# (namespace, SA 이름) ↔ IAM role 매핑을 AWS 쪽(PodIdentityAssociation)에서 관리
```

- **IRSA**: OIDC 기반. 클러스터의 OIDC provider를 IAM에 신뢰 관계로 등록하고, SA에 role ARN을
  annotation으로 명시. 매니페스트만 보면 어떤 role을 쓰는지 알 수 있다
- **Pod Identity**: 매핑을 AWS 쪽에서 관리(Terraform의 `aws_eks_pod_identity_association`).
  매니페스트에는 아무 표시가 없어서, **role을 확인하려면 인프라 코드를 봐야 한다**
- 실제로 한 클러스터에 둘이 섞여 있을 수 있다. 추천 API는 Pod Identity를, Airflow와 Grafana는
  IRSA annotation을 쓴다 (참고 4번)

## 3. 대안

- **설정을 넣는 방법**

|방식|버전 관리|갱신|비밀|어울리는 곳|
|---|---|---|---|---|
|이미지에 굽기|가능|재빌드 필요|불가|바뀌지 않는 기본값|
|ConfigMap|가능(git)|볼륨은 자동|불가|환경별 설정 파일|
|Secret (직접 생성)|**불가**(비밀)|수동|가능|외부 저장소가 없을 때|
|CSI + 외부 저장소|가능(참조만)|저장소에서 회전|가능|운영 환경 자격증명|

  - CSI 방식의 핵심 이점은 **git에 값이 아니라 "어디서 가져올지"만 남는다**는 것

- **AWS 자격증명 부여 방식**

|구분|IRSA|Pod Identity|
|---|---|---|
|매핑 위치|SA annotation (매니페스트)|AWS 쪽 association|
|매니페스트 가독성|role 이 보임|보이지 않음|
|클러스터 간 재사용|OIDC provider 별로 신뢰 설정|association 만 다시 만들면 됨|
|설정 지점|k8s + IAM 양쪽|주로 AWS 쪽|

  - 어느 쪽이든 **Pod에 SA를 지정하는 것을 잊으면 조용히 실패한다**. `serviceAccountName`이
    없으면 `default` SA가 쓰이고, 그 SA에는 아무 권한이 없다

## 4. 참고

**1. SecretProviderClass를 만들었는데 Secret이 생기지 않는다**

가장 흔한 실수다. `secretObjects`는 "마운트되면 이런 Secret을 만들어라"는 선언일 뿐,
**Pod이 CSI 볼륨을 실제로 마운트해야** 동기화가 일어난다.

- 그래서 Secret을 `env`로만 쓰고 볼륨은 안 붙인 Pod은 부팅에 실패한다 — 참조하는 Secret이
  존재하지 않기 때문
- Airflow에서는 모든 컴포넌트에 CSI 볼륨을 붙여 이 문제를 피했다. 렌더로 확인 가능한 부분이다

  ```text
  CSI 볼륨이 마운트된 워크로드: scheduler, api-server, dag-processor,
                              triggerer, cleanup, db-cleanup
  ```

- 순서를 정리하면: **볼륨 마운트 → Secret 생성 → 다른 Pod이 그 Secret 참조**

**2. 값이 없는 placeholder는 어디서는 안전하고 어디서는 렌더를 깨뜨린다**

values에 값을 비워두는 관례(`ssl-certificate:    # 인증서 위치`)를 쓸 때, 그 값이
**어디에 쓰이는지**에 따라 결과가 다르다.

|위치|null 일 때 결과|
|---|---|
|annotation map 값|해당 항목이 조용히 **탈락** (안전)|
|문자열 보간(이미지 이름 등)|`image: %!s(<nil>):latest` → **YAML 파싱 실패**|

- 전자는 Ingress annotation에서 확인했다. 값이 없는 세 개는 렌더 결과에서 사라지고
  값이 있는 것만 남는다
- 후자는 Airflow의 `defaultAirflowRepository`를 비웠을 때 겪었다. 이미지 이름은 문자열로
  조립되므로 null이 그대로 문자열에 박혀 렌더가 깨진다 → 이런 값은 반드시 더미 문자열을 둔다

**3. 시크릿 전달 체인은 고리가 많아 어디서 끊겼는지 찾기 어렵다**

Grafana의 Google OAuth를 예로 들면 고리가 다섯 개다.

```text
Secrets Manager (prod/grafana/oauth, JSON)
  → SecretProviderClass  : jmesPath 로 client_id / client_secret 추출
  → Pod 의 CSI 마운트     : 이때 동기화 발동
  → k8s Secret 생성       : grafana-google-oauth
  → envFromSecret        : 컨테이너 env 로 주입
  → grafana.ini          : ${GF_AUTH_GOOGLE_CLIENT_ID} 참조
```

- 어느 고리가 끊겼는지 확인하는 순서: SPC가 렌더됐나 → Pod에 볼륨이 붙었나 → Secret이
  생성됐나 → env에 들어왔나 → 앱이 그 이름으로 읽나
- 이름을 한 곳에서만 바꾸면 조용히 실패한다. 특히 `secretObjects`의 `key`와 앱이 기대하는
  env 이름은 별개라 둘을 맞춰야 한다

**4. 같은 클러스터에서 IRSA와 Pod Identity가 섞일 수 있다**

추천 API의 ServiceAccount에는 annotation이 없고, Airflow의 것에는 role ARN이 있다.
매니페스트만 보면 전자는 "권한이 없어 보이는데" 실제로는 Pod Identity로 부여된다.

- **판단 방법**: SA에 annotation이 없는데 AWS API 호출이 성공한다면 Pod Identity를 의심한다.
  매핑은 인프라 코드(Terraform)에 있다
- 새 워크로드를 추가할 때는 그 클러스터의 관례를 먼저 확인해야 한다. IRSA 방식 클러스터에
  annotation 없는 SA를 만들면 권한이 없고, 반대의 경우 annotation이 무시된다

**5. ConfigMap 이름을 릴리스 이름으로 조립할 때 주의**

커스텀 템플릿에서 차트가 만든 ConfigMap을 참조할 때, 이름을 직접 조립하면 릴리스 이름에
의존하게 된다.

```yaml
volumes:
  - name: config
    configMap:
      name: {{ .Release.Name }}-config     # airflow-config
```

- 릴리스 이름이 `airflow`가 아니면 실제 ConfigMap 이름과 어긋나 마운트가 실패한다
- ArgoCD Application에서 `releaseName`을 고정해 이 위험을 없앤다

  ```yaml
  helm:
    releaseName: airflow
  ```

- 더 안전한 방법은 차트가 제공하는 fullname 헬퍼를 쓰는 것이지만, wrapper 차트에서는
  subchart의 헬퍼를 그대로 쓸 수 없는 경우가 있어 릴리스 이름 고정이 현실적인 타협이다
