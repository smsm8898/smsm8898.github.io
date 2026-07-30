---
title: "AWS Infrastructure as Code"
date: 2026-07-28T18:00:00+09:00
categories: [Infrastructure]
tags: [Infrastructure, Terraform, AWS, VPC, EKS, EC2, ECR, IAM]
summary: "Terraform으로 AWS 인프라를 한 조각씩 쌓아가며 정리하는 시리즈 개요 — provider·state 같은 도구의 뼈대부터 VPC·EKS·EC2·ECR·IAM까지, 의존성 순서대로."
series: [AWS Infrastructure as Code]
---

이 글은 **AWS Infrastructure as Code** 시리즈의 출발점입니다. EKS 기반 인프라를 도메인
단위로 한 조각씩 쌓아보면서, 각 AWS 리소스가 왜 그렇게 설정되어 있는지를 한 편씩
정리합니다.

## 1. 왜 필요한가?
- **배경(콘솔 클릭의 한계)**: AWS 콘솔에서 클릭으로 만든 인프라는 "지금 어떻게 생겼는지"를 아무도 정확히 모름. 누가 언제 왜 바꿨는지 이력이 없고, 같은 구성을 dev/prod에 재현할 방법이 없음
- **해결책(IaC)**: 인프라를 코드로 선언해두면 ① 코드 리뷰로 변경을 검증하고 ② git 히스토리가 곧 변경 이력이 되고 ③ 같은 코드에 값만 바꿔 다른 환경을 복제할 수 있음
- **왜 Terraform인가**: AWS 전용(CloudFormation)이 아니라 provider 플러그인 구조라 AWS·Kubernetes·Helm을 한 코드베이스에서 다룸. EKS 위에 Helm 릴리스까지 이어지는 구성에서 특히 유리

## 2. 무엇을 다루는가?
- **도구의 뼈대**: provider(누가 어디에), state(무엇을 만들었는지 기록), 변수(환경별로 달라지는 값)
- **AWS 리소스**: 네트워크(VPC·Subnet·SG) → 컴퓨트(EKS·Node Group·EC2) → 권한(IAM·Pod Identity) → 스토리지·레지스트리(S3·ECR) → DNS(Route53)
- **학습 방식**: 실제 `apply`는 하지 않고 **`terraform validate` 통과를 각 단계의 성공 기준**으로 삼음. 계정 번호·리소스 ID·도메인은 예시 값을 쓴다

## 3. 구성 요소

### 3-1. Terraform 쪽 개념

| 조각 | 역할 |
|---|---|
|provider|AWS·Kubernetes 같은 대상을 다루는 법을 아는 플러그인. 어느 계정·리전에 만들지 결정|
|state|"코드상의 이름 ↔ 실제 AWS 리소스"를 매핑해 기억하는 JSON. 없으면 이미 만든 걸 또 만들려 함|
|backend|state를 어디에 보관할지. 팀·CI가 공유하려면 S3 같은 원격 저장소가 필요|
|variables / tfvars|환경별로 달라지는 값을 코드에서 분리. 같은 코드로 dev와 prod를 만드는 장치|
|lock file|설치된 provider 버전과 체크섬 고정. `package-lock.json`과 같은 역할|

### 3-2. AWS 리소스 쪽

| 조각 | 역할 |
|---|---|
|VPC · Subnet|인프라가 들어갈 사설 네트워크와 그 안의 구획(public/private)|
|Security Group|리소스 단위 방화벽. "누가 어떤 포트로 들어올 수 있나"|
|EKS Cluster|관리형 Kubernetes 컨트롤 플레인|
|Node Group|워커 노드(EC2) 묶음. 워크로드가 실제로 돌아가는 곳|
|IAM · Pod Identity|누가 무엇을 할 수 있는지. 파드가 AWS API를 호출하려면 파드에도 역할이 필요|
|ECR|컨테이너 이미지 저장소|
|S3|로그·아티팩트·Terraform state 보관|
|Route53|도메인을 로드밸런서로 연결|

## 4. 구체적 사례

HCL 문법과 state 개념이 처음이라면 [Terraform · HCL · State](../basic/)를 먼저 읽는 것이 좋다.
도구가 어떻게 동작하는지 알고 나면 아래 각 편의 코드가 훨씬 빨리 읽힌다.

의존성 순서대로 쌓는다. 앞 편이 만든 것을 뒤 편이 참조하는 구조라, 순서를 바꾸면
참조할 리소스가 없어 `validate`부터 실패한다.

1. [Provider · Backend · Variables](../bootstrap/) : 리소스 0개로 판 깔기 — 어느 계정에 만들지, state를 어디 둘지, 환경별 값을 어떻게 분리할지
2. VPC · Subnet · Security Group *(예정)* : 모든 리소스가 올라갈 네트워크
3. EKS Cluster *(예정)* : 컨트롤 플레인과 인증 구조
4. Node Group *(예정)* : 워커 노드, taint/label로 워크로드를 가르는 법
5. IAM · Pod Identity *(예정)* : 클러스터·노드 역할과 파드 단위 AWS 권한
6. ECR · S3 *(예정)* : 이미지 레지스트리와 버킷 수명주기
7. Route53 *(예정)* : 내부/외부 도메인과 cross-account 위임
8. Atlantis *(예정)* : PR에 plan을 붙이고 승인 후 apply하는 자동화

## 참고
- 이 시리즈는 [Observability Pipeline](../../observability/overview/) 시리즈와 짝을 이룬다. 여기서 만든 EKS 위에 그쪽 모니터링 스택이 올라간다
