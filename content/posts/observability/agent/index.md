---
title: "Agent"
date: 2026-07-24
tags: ["Agent", "Alloy"]
categories: ["Observability"]  
summary: "Metrics, Logs, Traces 수집기를 하나로 합친 차세대 올인원 에이전트 Grafana Alloy의 도입 배경과 설정 방법"
---

## 1. 정의
- 애플리케이션과 인프라에 흩어진 데이터를 주워 담아 중앙 저장소로 배달

## 2. 종류
- Prometheus
  - 직접 수집 -> Prometheus
  - 역할 및 정의: 별도의 에이전트 없이 Prometheus 서버가 주기적으로 타겟을 방문해 metrics을 직접 당겨온다(`Pull`)
  - 장점: `PodMonitor` 등을 통한 k8s native 설정이 직관적이며 추가 에이전트가 필요 없다
  - 단점: 메트릭 전용
- Promtail
  - Agent -> Loki
  - 역할 및 정의: 파드의 stdout 파일 등을 읽어 **로그(글자)**를 수집한 뒤 로키로 밀어 넣는다(`Push`)
  - 장점: 설정이 매우 단순하고 가벼우며 Loki와의 연동 및 라벨링 처리가 완벽
  - 단점: 로그전용
- OpenTelemetry Collector
  - Agent -> Tempo
  - 역할 및 정의: 벤더에 종속되지 않는 범용 수집기. 주로 trace 데이터를 받아 tempo로 전송
  - 장점: Datadog, Grafana 등 특정 벤더에 종속되지 않는 범용 표준
  - 단점: 초기 아키텍처 구성과 YAML 설정이 복잡하여 진입 장벽이 높다
  
## 3. 대안
- Grafana Alloy
  - 기존 문제점
    - 관리의 파편화: 노드마다 Promtail, OTel 등 여러 개의 데몬을 띄워야 하고 유지보수의 어려움
    - 리소스 오버헤드: 수집들이 각각 별도의 CPU & Mem 점유
    - 데이터 연계 단절: "로그에서 발견된 에러를 메트릭 수치로 변환해서 카운트" 같은 파이프라인 구성이 어려움
  - 해결
    - 올인원: 메트릭(Pull), 로그(Push), 트레이스(Receive)를 단 하나의 데몬으로 통합하여 리소스 낭비 최소화
    - 단일 설정 언어: 고유의 파이프라인 조립 문법(Alloy Syntax) 하나로 모든 데이터 수집·가공
    - 라우팅 파이프라인: 수집한 데이터를 필터링하고 변환한 뒤 Prometheus, Loki, Tempo로 알아서 분배

## 4. 참고
1. Grafana Alloy vs OpenTelemetry Collector (다를 게 뭐야?)
- Alloy의 핵심 엔진(코어)은 OpenTelemetry(OTel) Collector 자체
- 하지만 '그라파나 생태계(Prometheus, Loki)와의 완벽한 호환성

|구분|OpenTelemetry Collector|Grafana Alloy|
|---|---|---|
|태생|특정 벤더(회사)에 종속되지 않는 범용 표준|순정 OTel의 장점 + 그라파나 생태계 맞춤형 튜닝|
|설정 언어|아주 길고 복잡한 YAML|프로그래밍 언어와 유사한 HCL (블록 조립)|
|Prometheus 호환성|쿠버네티스의 PodMonitor 등을 읽으려면 설정이 꽤 까다로움|기존 kube-prometheus-stack의 PodMonitor를 100% 네이티브로 자동 인식함|
|Loki 호환성|OTel 로그를 Loki 형식으로 변환하는 과정에서 라벨링이 꼬이거나 손실될 우려가 있음|Promtail의 핵심 코드를 그대로 내장하여 완벽하게 연동됨|

2. Tempo는 왜 전용 수집기(Agent)가 없을까?
- 과거 분산 추적 도구들(Jaeger, Zipkin)은 jaeger-agent처럼 각자의 전용 수집기 존재
- 하지만 그라파나가 Tempo를 개발할 시점에는 이미 시장의 표준이 OpenTelemetry로 완전히 굳어진 상태

