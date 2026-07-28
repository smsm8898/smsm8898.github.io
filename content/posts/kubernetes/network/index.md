---
title: "Network"
date: 2026-07-28T22:00:00+09:00
tags: ["Kubernetes", "Service", "ClusterIP", "Headless", "Ingress", "ALB"]
categories: ["Kubernetes"]
summary: "Pod에 어떻게 접근하는가 — Service 세 종류(ClusterIP·headless·엔드포인트 없는 Service)의 차이, targetPort를 이름으로 참조하는 이유, Ingress 하나를 여러 서비스가 공유하는 ALB group 패턴"
---

워크로드가 떴다면 다음 질문은 **어떻게 접근하는가**다. 이번 글은 Service와 Ingress 편.

## 1. 정의

- **문제**: Pod의 IP는 재생성마다 바뀌고, replicas가 여러 개면 어디로 보내야 할지도 정해야 한다.
  Pod IP를 직접 부르는 코드는 존재할 수 없음
- **Service**: 라벨로 Pod들을 묶어 **고정된 이름과 가상 IP**를 부여한다. 클라이언트는
  `product-personalize-api:8000`처럼 이름으로 부르고, Service가 살아있는 Pod 중 하나로 넘긴다
- **Endpoints(EndpointSlice)**: Service가 실제로 트래픽을 넘길 Pod IP 목록. Service가 직접
  관리하지 않고, **selector에 맞고 readiness를 통과한 Pod**이 자동으로 등록된다
  - 즉 readinessProbe가 실패하면 Pod은 살아있어도 이 목록에서 빠진다 → 워크로드 편의 readiness가
    여기서 효과를 낸다
- **Ingress**: L7(HTTP) 경로 기반 라우팅 규칙. 규칙만 선언하고, 실제 로드밸런서는
  **Ingress Controller**가 만든다 (EKS에서는 AWS Load Balancer Controller가 ALB를 생성)

## 2. 종류

### 2-1. ClusterIP — 기본값, 클러스터 내부 전용

```yaml
apiVersion: v1
kind: Service
metadata:
  name: product-personalize-api
spec:
  type: ClusterIP
  selector:
    app-name: product-personalize-api    # 이 라벨을 가진 Pod 들로 보낸다
  ports:
    - name: service-port
      port: 8000                         # Service 가 노출하는 포트
      targetPort: deploy-port            # Pod 의 포트 — 숫자가 아니라 이름
```

- **`targetPort`를 이름으로 쓰는 이유**: Pod 명세에서 `ports: [{name: deploy-port, containerPort: 8000}]`로
  이름을 붙여두면, 컨테이너 포트가 바뀌어도 Service를 고칠 필요가 없다. 숫자를 두 곳에 적으면
  한쪽만 바꿨을 때 조용히 연결이 끊긴다
- `port`와 `targetPort`가 다를 수 있다는 점이 핵심이다. Service는 8000으로 받아 Pod의
  다른 포트로 넘길 수 있음
- ClusterIP는 클러스터 밖에서 접근할 수 없다. 외부 노출은 Ingress(L7) 또는 LoadBalancer 타입(L4)의 몫

### 2-2. Headless Service — 개별 Pod을 지목해야 할 때

```yaml
spec:
  clusterIP: None      # ← 가상 IP 를 만들지 않는다
  selector:
    component: triggerer
```

- **가상 IP도 로드밸런싱도 없다.** DNS 조회 결과로 **Pod IP 목록을 그대로** 돌려준다
- StatefulSet의 짝이다. `spec.serviceName`에 headless Service를 지정하면 각 Pod에 고유 DNS가 생긴다

  ```text
  airflow-triggerer-0.airflow-triggerer.airflow.svc.cluster.local
  <pod>-<ordinal>.<service>.<namespace>.svc.cluster.local
  ```

- 필요한 상황: 클러스터 멤버가 서로를 개별적으로 찾아야 하는 앱(DB 복제, 분산 캐시), 또는
  Spark driver가 executor와 직접 통신하는 경우

### 2-3. 엔드포인트 없는 Service — 트래픽이 아니라 "대상 표시"

트래픽을 넘기지 않는데도 존재하는 Service가 있다.

```text
monitoring-coredns
monitoring-kube-controller-manager
monitoring-kube-etcd
monitoring-kube-proxy
monitoring-kube-scheduler
```

kube-prometheus-stack이 만드는 이 Service들은 아무도 호출하지 않는다. **Prometheus가
스크랩할 대상을 가리키기 위한 표지판**이다. ServiceMonitor가 Service를 셀렉터로 찾고,
그 Service의 엔드포인트(= 실제 Pod들)를 스크랩 대상으로 삼는 구조다.

- **관리형 컨트롤 플레인에서는 이게 빈다**: EKS는 controller-manager·scheduler·etcd를
  사용자에게 노출하지 않는다. Service는 만들어지지만 selector에 맞는 Pod이 없어
  **엔드포인트가 0개**가 되고, Prometheus 입장에서는 영원히 스크랩 실패하는 dead target이 된다
  → 그런 컴포넌트는 차트에서 아예 끄는 것이 맞다 (참고 3번)

### 2-4. Ingress — 경로 기반 외부 노출

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/scheme: internal
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/group.name: dev-internal-ingress   # ← 여러 Ingress 가 ALB 하나를 공유
    alb.ingress.kubernetes.io/healthcheck-path: /health
spec:
  ingressClassName: alb
  rules:
    - host: reco-api.dev.example.com
      http:
        paths:
          - path: /api/v1/products/personalize
            pathType: ImplementationSpecific
            backend:
              service:
                name: product-personalize-api
                port: { name: service-port }
```

- **`spec`은 표준, `annotations`는 컨트롤러 방언**: 경로·호스트·백엔드는 Kubernetes 표준
  스펙이지만, TLS 인증서·스킴·헬스체크 같은 클라우드별 설정은 전부 annotation으로 들어간다.
  ALB용 차트를 다른 컨트롤러(nginx)로 옮기면 annotation을 전부 다시 써야 하는 이유
- **`target-type: ip` vs `instance`**: `ip`는 ALB가 Pod IP로 직접 보낸다(VPC CNI 전제).
  `instance`는 NodePort를 거치므로 홉이 하나 늘고 클라이언트 IP 보존이 까다롭다
- **`group.name`이 비용을 줄인다**: ALB는 개당 과금이라 Ingress마다 새로 만들면 낭비다.
  같은 `group.name`을 가진 Ingress들은 **하나의 ALB를 공유**하고, `group.order`로 규칙 평가
  순서를 정한다
- **path가 여러 서비스로 갈린다**: 추천 API는 서비스 3개가 각자 다른 경로를 가지되
  Ingress는 하나다. `/api/v1/products/personalize` → 개인화 서비스,
  `/api/v1/products/related` → 연관상품 서비스

## 3. 대안

- **외부 노출 방식**

|방식|계층|과금 단위|어울리는 곳|
|---|---|---|---|
|`ClusterIP` + Ingress|L7 (HTTP)|ALB 1개를 여러 서비스가 공유|HTTP API, 웹 UI (대부분의 경우)|
|`type: LoadBalancer`|L4 (TCP)|서비스마다 NLB/CLB 1개|gRPC·TCP 프로토콜, HTTP가 아닌 것|
|`type: NodePort`|L4|없음(노드 포트 직접)|로컬 실험, Ingress 컨트롤러 자체 노출|

  - Service 타입은 계단식이다. LoadBalancer는 NodePort를 포함하고, NodePort는 ClusterIP를 포함한다

- **DNS 조회 결과 비교**

|구분|ClusterIP|Headless (`clusterIP: None`)|
|---|---|---|
|DNS 응답|가상 IP 1개|Pod IP 전부|
|로드밸런싱|kube-proxy가 분배|클라이언트가 직접 선택|
|Pod 개별 지목|불가|가능 (`<pod>-0.<svc>`)|
|어울리는 곳|상태 없는 API|StatefulSet, 클러스터링 앱|

- **Pod 노출이 Service만 있는 건 아니다**: Prometheus는 Service 없이도 Pod을 직접
  스크랩할 수 있다(PodMonitor). "트래픽을 받는 것"과 "관측 대상이 되는 것"은 별개의 문제다

|구분|ServiceMonitor|PodMonitor|
|---|---|---|
|대상 탐색|Service → 엔드포인트|Pod 라벨 직접|
|Service 필요|필요|불필요|
|어울리는 곳|이미 Service가 있는 컴포넌트|Service를 만들 이유가 없는 워커·사이드카|

## 4. 참고

**1. Service 이름으로 호출하는 것이 왜 되는가**

클러스터 안의 Pod은 CoreDNS를 DNS 서버로 쓰고, `/etc/resolv.conf`에 검색 도메인이 들어있다.
그래서 같은 네임스페이스에서는 짧은 이름으로도 찾아진다.

```text
product-personalize-api                              # 같은 네임스페이스
product-personalize-api.reco                         # 다른 네임스페이스
product-personalize-api.reco.svc.cluster.local       # FQDN
```

- 이 구조 때문에 **CoreDNS가 죽으면 서비스 간 통신이 연쇄로 끊긴다**. 모니터링에서
  `CoreDNSDown`을 critical로 두는 이유
- 반대로 CoreDNS 자체의 스크랩이 깨져도 알림은 조용하다. 값이 없으면 임계치 비교가
  일어나지 않으므로, "메트릭 소멸" 감시(`absent()`)를 따로 걸어야 한다

**2. 무중단 배포는 Service만으로 완성되지 않는다**

`maxUnavailable: 0`으로 새 Pod을 먼저 띄워도 504가 날 수 있다. Pod 종료 시
**엔드포인트 제거와 SIGTERM이 동시에** 일어나고, ALB의 디레지스터는 그보다 느리기 때문이다.

```yaml
lifecycle:
  preStop:
    exec: { command: ["/bin/sh", "-c", "sleep 30"] }
terminationGracePeriodSeconds: 60      # preStop(30) + 앱 종료 시간보다 커야 한다
```

- ALB가 이 Pod을 대상에서 뺄 때까지 30초를 버티고, 그 사이 들어온 요청은 정상 응답한다
- grace period가 preStop보다 짧으면 sleep 중에 SIGKILL을 맞아 오히려 요청이 끊긴다

**3. 엔드포인트가 0개인 Service는 조용한 장애를 만든다**

관리형 EKS에서 `kubeControllerManager`·`kubeScheduler`·`kubeEtcd`·`kubeProxy` 스크랩을
켜두면, Service는 만들어지지만 대상 Pod이 없어 엔드포인트가 비게 된다.

```yaml
# prod values — 관리형 컴포넌트는 명시적으로 끈다
kubeControllerManager: { enabled: false }
kubeScheduler:         { enabled: false }
kubeEtcd:              { enabled: false }
kubeProxy:             { enabled: false }
```

- 끄지 않으면 Prometheus 타겟 목록에 영구 실패 항목이 남고, `TargetDown` 같은 알림이
  상시 발화해 노이즈가 된다. 알림을 무음 처리하는 것보다 **대상 자체를 없애는 것**이 맞다

**4. Service의 selector와 워크로드의 라벨은 별개로 관리된다**

Service는 컨트롤러를 모른다. **라벨만 본다.** 그래서 Deployment를 지우고 다시 만들어도
라벨이 같으면 Service는 그대로 동작하고, 반대로 라벨을 한 글자 바꾸면 Service는
엔드포인트가 0인 채로 조용히 살아있다.

- 이 느슨한 연결이 유용한 경우: blue/green 배포에서 Service의 selector만 바꿔 트래픽을 전환
- 위험한 경우: 차트에서 라벨 헬퍼를 수정했는데 Service selector는 그대로 → 연결이 끊기지만
  에러는 없다. kube-prometheus-stack에서 CoreDNS의 ServiceMonitor selector가 요구하는 라벨이
  Service에 붙지 않아 타겟이 0개가 된 사고가 이 종류다

**5. Ingress 하나에 여러 서비스를 붙일 때 순서가 중요하다**

경로 규칙은 위에서부터 평가되므로, 넓은 경로를 먼저 두면 좁은 경로가 도달하지 못한다.

```yaml
paths:
  - path: /api/v1/products/personalize   # 구체적인 것 먼저
  - path: /api/v1/products               # 넓은 것 나중
```

- ALB group을 공유하는 **다른 Ingress와의 순서**는 `group.order`로 정한다. 값이 작을수록 먼저
  평가되므로, catch-all 규칙을 가진 Ingress에는 큰 값을 준다
