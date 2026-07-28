---
title: "Kubernetes Resources"
date: 2026-07-28T20:00:00+09:00
categories: [Kubernetes]
tags: [Kubernetes, Helm, Workload, Service, ConfigMap, Secret, CRD, Operator]
summary: "Helm 차트 세 개를 직접 작성하며 만난 Kubernetes 리소스를 계층별로 정리하는 시리즈 개요 — 워크로드·네트워크·설정·스케줄링·오토스케일링·CRD까지, 렌더 결과를 읽는 방식으로."
series: [Kubernetes Resources]
---

이 글은 **Kubernetes Resources** 시리즈의 출발점입니다. 애플리케이션 하나(추천 API),
모니터링 스택(kube-prometheus-stack), 워크플로 엔진(Airflow)의 Helm 차트를 직접 작성하면서
만난 리소스들을 계층별로 한 편씩 정리합니다.

## 1. 왜 필요한가?

- **배경(Helm은 리소스를 감춘다)**: `helm install`만 하면 파드가 뜬다. 리소스를 몰라도 동작하므로
  values 파일의 키만 만지다가, 문제가 생기는 순간 막힌다. "왜 파드가 Pending인가", "왜 대시보드에
  데이터가 없나", "왜 알림이 안 오나" 같은 질문의 답은 values가 아니라 **렌더된 리소스**에 있음
- **해결책(렌더 결과를 읽는다)**: `helm template`으로 실제로 어떤 리소스가 어떤 필드로 만들어지는지
  확인하는 습관. 차트는 결국 리소스를 만드는 도구이고, 디버깅은 리소스 층에서 이뤄짐
- **왜 이 세 프로젝트인가**: 성격이 달라서 커버 범위가 겹치지 않는다. 상태 없는 API는 기본 리소스를,
  모니터링 스택은 CRD·Operator를, 워크플로 엔진은 상태 있는 워크로드와 동적 파드 생성을 보여줌

## 2. 무엇을 다루는가?

- **리소스 계층**: 워크로드(무엇을 실행하나) → 네트워크(어떻게 접근하나) → 설정·시크릿(값을 어떻게
  넣나) → 스케줄링(어디서 실행하나) → 오토스케일링(몇 개로 실행하나) → CRD·Operator(리소스를
  직접 만들어 쓰는 법)
- **학습 방식**: 실제 클러스터에 배포하지 않고 **`helm template` 렌더 결과와 `helm lint` 통과를
  각 단계의 성공 기준**으로 삼음. 계정 ID·도메인·ECR 주소는 전부 더미로 치환하고, 커밋마다
  실값이 새지 않는지 grep으로 검사
- **차별점은 함정**: 리소스의 정의와 필드는 공식 문서에 다 있다. 대신 각 편 마지막에
  직접 만들어보며 겪은 함정을 남긴다 — 렌더에 안 보이는 리소스, 조용히 죽는 알림, 영구 Pending 같은 것들

## 3. 구성 요소

### 3-1. 리소스 계층

| 계층 | 리소스 | 답하는 질문 |
|---|---|---|
|워크로드|Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob|무엇을, 어떻게 실행하나|
|네트워크|Service(ClusterIP·headless), Ingress|어떻게 접근하나|
|설정·시크릿|ConfigMap, Secret, SecretProviderClass(CSI), ServiceAccount|값과 자격증명을 어떻게 넣나|
|스케줄링|nodeSelector, taint·toleration, QoS, affinity|어느 노드에서 실행하나|
|오토스케일링|HPA, PodDisruptionBudget, replicas|몇 개로 실행하고, 몇 개까지 죽어도 되나|
|확장|CRD, Operator, ServiceMonitor·PodMonitor·PrometheusRule|리소스 종류를 직접 만들어 쓰는 법|
|권한|Role·RoleBinding, ClusterRole|누가 무엇을 할 수 있나|

### 3-2. 세 프로젝트가 커버하는 범위

| 리소스 | 추천 API | kube-prometheus-stack | Airflow |
|---|---|---|---|
|Deployment|✓ (서비스 3개)|✓ (grafana, operator)|✓ (scheduler, api-server, dag-processor)|
|StatefulSet| |△ (CR로 위임)|✓ (triggerer)|
|DaemonSet| |✓ (node-exporter)| |
|Job · CronJob| | |✓ (migration, cleanup, db-cleanup)|
|동적 Pod 생성| | |✓ (KubernetesExecutor task pod)|
|Service · Ingress|✓ (ALB)|✓ (grafana)|✓ (api-server)|
|ConfigMap| |✓ (대시보드 JSON)|✓ (airflow.cfg, pod template)|
|Secret · CSI|✓|✓ (Grafana OAuth)|✓ (DB 접속, fernet key, deploy key)|
|HPA · PDB|✓| | |
|CRD · Operator| |✓ (핵심)| |
|RBAC| | |✓ (pod launcher, spark)|

한 리소스를 여러 프로젝트에서 다르게 쓰는 경우가 학습에 가장 유용하다. 예를 들어 Secret은
추천 API에서는 앱 자격증명, 모니터링 스택에서는 OAuth, Airflow에서는 DB 접속 문자열과
git deploy key로 쓰이고, 세 곳 모두 AWS Secrets Manager에서 CSI로 가져오는 같은 구조를 쓴다.

## 4. 구체적 사례

계층 순서대로 쌓는다. 앞 편의 리소스를 뒤 편이 참조하는 구조다 — 워크로드가 있어야 노출할
대상이 있고, 그 다음에야 스케줄링과 오토스케일링을 말할 수 있다.

1. [워크로드](../workload/) : Pod와 컨트롤러 6종. 무엇을 언제 쓰나, 그리고 직접 만들지 않고 위임하는 두 가지 방식
2. [네트워크](../network/) : ClusterIP와 headless의 차이, Ingress에서 ALB로 이어지는 경로
3. [설정·시크릿](../config-secret/) : ConfigMap의 여러 마운트 방식, Secrets Manager → CSI → Secret → env 체인
4. [스케줄링](../scheduling/) : nodeSelector만으로 부족한 이유(taint), QoS 등급이 축출 순서를 바꾸는 방식
5. [오토스케일링 · 중단](../autoscaling/) : HPA와 replicas의 충돌, PDB가 노드 작업을 막는 방식
6. [CRD · Operator](../crd-operator/) : 리소스 종류를 추가하는 법과, 그 리소스가 다시 워크로드를 만드는 구조

## 참고

- 이 시리즈는 [AWS Infrastructure as Code](../../infrastructure/overview/) 시리즈와 짝을 이룬다.
  그쪽이 EKS 클러스터와 노드그룹을 만들고, 이 시리즈는 그 위에 워크로드를 올린다. 스케줄링 편의
  taint·label은 노드그룹 편에서 만든 것을 그대로 쓴다
- [Observability Pipeline](../../observability/overview/) 시리즈가 관측성의 *이론*(무엇을 왜 측정하나)이라면,
  이 시리즈의 CRD 편은 그 이론이 *어떤 리소스로 구현되는지*를 다룬다. Prometheus가 스크랩 대상을
  아는 방법이 ServiceMonitor·PodMonitor라는 CRD다
