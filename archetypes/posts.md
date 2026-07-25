---
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
date: {{ .Date }}
tags: []         # 다루는 기술·기법 이름 (Hacker News, Airflow, Prometheus, …)
# categories: []  # (선택) 넓게 묶고 싶을 때만
summary: ""
draft: true
---

<!-- 어떤 시리즈에 어떤 부분인지 간략 배경 -->

## 1. 정의
- 

## 2. 종류
- Prometheus
  - 직접 수집 -> Prometheus
  - 역할 및 정의: 
  - 장점: 
  - 단점: 
- Promtail
  - Agent -> Loki
  - 역할 및 정의: 
  - 장점: 
  - 단점: 
  
## 3. 대안
- Grafana Alloy
  - 기존 문제점
    - 
  - 해결
    - 

## 4. 참고
1. Grafana Alloy vs OpenTelemetry Collector (다를 게 뭐야?)
- Alloy의 핵심 엔진(코어)은 OpenTelemetry(OTel) Collector 자체
- 하지만 '그라파나 생태계(Prometheus, Loki)와의 완벽한 호환성

|구분|OpenTelemetry Collector|Grafana Alloy|
|---|---|---|
||||


2. Tempo는 왜 전용 수집기(Agent)가 없을까?
- 

