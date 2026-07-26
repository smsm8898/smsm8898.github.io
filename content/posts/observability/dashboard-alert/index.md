---
title: "Dashboard & Alert"
date: 2026-07-26
tags: ["Grafana", "Alertmanager", "Robusta"]
categories: ["Observability"]
summary: "DB에 쌓인 데이터를 사람이 만나는 마지막 단계 — 직접 보러 가는 Grafana, 시스템이 사람을 부르는 Alertmanager, 그리고 알림에 컨텍스트를 붙여주는 Robusta"
---

DB에 쌓인 데이터가 드디어 사람을 만나는 단계. 이번 글은 시리즈의 마지막, Dashboard & Alert 편.

## 1. 정의
- Dashboard: 저장소의 데이터를 사람이 **보러 가는** 시각화 단계
- Alert: 이상 상황을 시스템이 사람에게 **알려주는** 단계
- 계속해서 대시보드를 보고있을 수 없으니, alert를 통해 자동으로 이상을 감지하고 dashboard를 통해 시각적으로 확인

## 2. 종류
- Grafana
  - Prometheus·Loki·Tempo -> 사람
  - 역할 및 정의: 여러 datasource를 하나의 대시보드로 묶어 시각화하는 도구(최종 화면)
  - 장점: 메트릭 패널에서 시간축을 맞춰 로그 패널로 파고들기, exemplar로 메트릭 -> 트레이스 점프 등 **세 데이터의 상관관계 탐색**이 네이티브
  - 단점: 보러 가야만 보인다 즉, 아무도 안 보고 있는 새벽 3시의 장애는 대시보드가 알려주지 않음
- Alertmanager
  - Prometheus -> Alertmanager -> Slack·Email·Webhook
  - 역할 및 정의: Prometheus가 발생시킨 알림을 받아 라우팅·그룹핑·중복제거·silence 처리하는 알림 전담 서버
  - 장점: 그룹핑으로 알림 폭탄 방지, 점검 시간엔 silence로 무음 처리, 라벨 기반으로 팀별 채널 라우팅
  - 단점: 알림이 텍스트뿐 — "에러율 5% 초과"라는 사실만 오고, 원인을 보려면 결국 사람이 Grafana를 열어야 함

## 3. 대안
- Robusta
  - 기존 문제점 (텍스트 알림의 한계)
    - 컨텍스트 부재: 알림은 "무슨 일이 났다"까지만 알려주고, "왜"는 받은 사람이 직접 조사해야 함
    - 대응 지연: 알림 확인 -> 노트북 열기 -> 클러스터 접속 -> `kubectl logs` 치는 과정 자체가 장애 시간
  - 해결
    - 컨텍스트 자동 첨부: Alertmanager의 알림을 webhook으로 가로채서, 해당 시점의 그래프·pod 로그·k8s 이벤트를 붙여 Slack으로 전달
    - 알림의 성격 전환: 알림이 '조사의 시작점'이 아니라 '조사 결과 요약'이 됨 — 채팅방에서 바로 원인 파악

## 4. 참고
1. 알림 규칙(Alert Rule)은 왜 Grafana가 아니라 Prometheus에 있나?
- 판정은 데이터가 있는 곳에서: Prometheus가 자신의 TSDB에 PromQL 규칙(예: `에러율 > 5%`)을 주기적으로 평가(evaluate)
- 역할 분담이 명확함 — Prometheus는 **"발생 판정"**, Alertmanager는 **"전달 방법"**(누구에게, 언제, 얼마나 묶어서)

2. Grafana Alerting vs Alertmanager (Grafana에도 알림 기능 있잖아?)

|구분|Alertmanager|Grafana Alerting|
|---|---|---|
|평가 주체|Prometheus가 규칙을 평가, AM은 전달만 담당|Grafana가 직접 datasource에 쿼리를 날려 평가|
|규칙 관리|YAML 파일 -> GitOps로 코드 관리에 유리|UI에서 클릭으로 생성 -> 시작이 쉬움|
|데이터 범위|Prometheus 메트릭 기반|Loki, Tempo 등 모든 datasource 기반 알림 가능|
|어울리는 곳|규칙을 코드로 관리하는 인프라 표준 구성|가볍게 시작하거나 로그 기반 알림이 필요할 때|

- 참고로 Grafana Alerting도 내부적으로는 Alertmanager를 내장해서 전달(라우팅·그룹핑)을 처리함

3. 시리즈 전체 흐름 복습 (금요일 오후 3시 장애 시나리오)
- 수집: Alloy가 앱의 메트릭·로그를 모아 Prometheus·Loki로 배달 (Agent 편)
- 감지: Prometheus가 알림 규칙 평가 중 에러율 5% 초과를 판정 (DB 편)
- 전달: Alertmanager가 라우팅 -> Robusta가 그래프와 pod 로그를 첨부해 Slack으로 전송
- 분석: 담당자가 Grafana에서 에러가 튄 구간을 확인, 시간 연동된 Loki 패널에서 원인 로그 발견
- 관측성 파이프라인의 모든 조각이 이 5분 안에 다 쓰임
