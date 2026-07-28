# Kubernetes 시리즈 설계 (2026-07-28)

## 배경

`infra-apps` repo에서 Helm 차트 3개를 직접 작성하며 얻은 것을 정리한다.

| 프로젝트 | 주로 다룬 리소스 |
|---|---|
| reco-api | Deployment/Service/Ingress/HPA/PDB/SA/PodMonitor/SecretProviderClass |
| kube-prometheus-stack | CRD·Operator, ServiceMonitor/PodMonitor/PrometheusRule, Prometheus CR |
| airflow | Deployment/StatefulSet/Job/CronJob, RBAC, 동적 생성 Pod(pod_template_file) |

기존 시리즈와의 관계:
- **AWS Infrastructure as Code**: EKS·노드그룹을 만드는 쪽. 이 시리즈는 그 위에 워크로드를 올린다
- **Observability Pipeline**: 관측성 *이론*. 이 시리즈는 그 이론이 어떤 k8s 리소스로 구현되는지

## 축: 리소스 계층별

프로젝트별이 아니라 리소스 계층별로 나눈다. 주인공은 Kubernetes 리소스이고,
3개 프로젝트는 각 리소스의 실제 사용 예시로 인용된다. 같은 리소스가 여러 글에
중복 등장하는 문제를 피하고, 나중에 Loki·Alloy를 해도 기존 글에 예시만 추가하면 된다.

```
content/posts/kubernetes/
├── overview/        # 리소스 지도, 세 프로젝트가 커버하는 범위
├── workload/        # Pod/Deployment/StatefulSet/DaemonSet/Job/CronJob
├── network/         # Service(ClusterIP·headless)/Ingress·ALB
├── config-secret/   # ConfigMap/Secret/CSI/IRSA
├── scheduling/      # nodeSelector/taint·toleration/QoS/affinity
├── autoscaling/     # HPA/PDB/replicas
└── crd-operator/    # CRD·Operator 패턴, ServiceMonitor/PodMonitor/PrometheusRule
```

frontmatter: `categories: [Kubernetes]`, `series: [Kubernetes Resources]`, tags는 리소스명.

## 글 구성 (기존 infrastructure 시리즈 스타일 준수)

```
## 1. 정의
## 2. 종류        # 2-1, 2-2 … 소절마다 핵심 필드 YAML + 불릿 설명
## 3. 대안        # 비교표 (언제 무엇을 쓰나)
## 4. 참고        # 번호별 Q&A. 직접 겪은 함정을 여기에 녹인다
```

infrastructure 시리즈에 있는 "직접 굴려보기"(재현 명령어) 항목은 두지 않는다 —
본문에서 이미 렌더 결과를 계속 인용하므로 중복이다.

이론은 공식 문서에도 있지만 함정은 직접 만들어봐야 얻으므로, 함정을 글의 차별점으로 둔다.

## 1차 산출물: overview + workload

나머지 5편은 톤을 확인한 뒤 같은 방식으로 진행.

### workload 편에 넣을 실제 사례

| 리소스 | 사례 |
|---|---|
| Deployment | reco-api 3개 서비스, airflow scheduler/api-server/dag-processor, grafana |
| StatefulSet | airflow triggerer / prometheus·alertmanager는 **CR로 위임** |
| DaemonSet | node-exporter (`tolerations: [operator: Exists]`) |
| Job | airflow migration·create-user (둘 다 껐음) |
| CronJob | airflow cleanup(15분), db-cleanup(주간) |
| 동적 Pod | KubernetesExecutor task pod (`pod_template_file`) |

### 함정 목록

1. StatefulSet인데 `helm template` 렌더에 없다 — Prometheus CR → operator가 런타임 생성
2. PVC 붙은 StatefulSet은 노드 재스케줄에서 교착 — dev emptyDir / prod PVC 분기
3. Job은 ArgoCD sync와 상성이 나쁘다 — migrateDatabaseJob·createUserJob을 끈 이유
4. CronJob 실패 pod을 지우면 알림이 죽는다 — `delete_worker_pods_on_failure: False`
5. DaemonSet은 taint를 전부 허용해야 한다 — 안 하면 특정 노드 메트릭이 조용히 빈다
6. Deployment 교체 전략 — RollingUpdate surge로 Pending 교착 → Recreate

## 검증

`hugo build` 통과, 상대 링크 유효성, 기밀 문자열(계정 ID·사내 도메인·ECR 주소) 미포함.
