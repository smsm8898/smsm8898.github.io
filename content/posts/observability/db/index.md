---
title: "DB"
date: 2026-07-25
tags: ["Prometheus", "Loki", "Tempo", "Mimir"]
categories: ["Observability"]
summary: "Metrics·Logs·Traces를 각각 저장하는 Prometheus·Loki·Tempo의 역할과 장단점, 그리고 장기 보관 문제를 해결하는 Mimir까지"
---

수집기(Alloy)가 모아온 데이터가 도착하는 곳. 이번 글은 저장소(DB) 편.

## 1. 정의
- 수집기가 배달한 Metrics, Logs, Traces를 받아 저장하고, 질의 언어(PromQL, LogQL, TraceQL)로 답하는 중앙 저장소

## 2. 종류
- Prometheus
  - Metrics -> PromQL
  - 역할 및 정의: 메트릭을 시간축 기준으로 압축 저장하는 시계열 DB(TSDB). 주기적으로 수집된 수치의 추세를 질의
  - 장점: k8s 모니터링의 사실상 표준. 강력한 PromQL과 Alertmanager 네이티브 연동
  - 단점: 단일 노드 설계라 수평 확장이 안 되고, 로컬 디스크 기반이라 장기 보관(수개월~수년)과 HA 구성이 어려움
- Loki
  - Logs -> LogQL
  - 역할 및 정의: "Prometheus, but for logs". 로그 본문은 인덱싱하지 않고 **label만 인덱싱**한 뒤 본문은 통째로 압축해 저장
  - 장점: 전문(full-text) 인덱스가 없어 저장 비용이 매우 저렴. label 체계가 Prometheus와 동일해서 메트릭 <-> 로그 전환이 자연스러움
  - 단점: 본문 검색은 사실상 grep 방식이라, 인덱스 기반 검색엔진 대비 넓은 범위의 전문 검색이 느림
- Tempo
  - Traces -> TraceQL
  - 역할 및 정의: 분산 추적(trace) 저장소. trace ID를 key로 요청의 전체 경로를 통째로 저장
  - 장점: 인덱스를 최소화하고 object storage에 바로 저장하는 구조라 대량의 trace를 싸게 보관. 샘플링 없이 100% 저장 가능
  - 단점: 태생이 trace ID 조회 전용이라, 조건 검색은 TraceQL(2.0+)이 나오고서야 가능해졌고 여전히 검색 기능은 상대적으로 약함

## 3. 대안
- Grafana Mimir
  - 기존 문제점 (Prometheus의 한계)
    - 확장 불가: 단일 노드 설계라 메트릭이 늘어나면 스케일 업 외에 답이 없음
    - 장기 보관: 로컬 디스크에 저장하므로 보존 기간을 늘리면 디스크가 감당 못함
    - HA 애매: 서버 2대를 띄우면 각자 따로 수집한 중복 데이터 2벌이 생길 뿐, 진짜 클러스터가 아님
  - 해결
    - 수평 확장: 저장을 여러 노드에 분산해 수억 개 시계열까지 스케일 아웃
    - Object Storage: 데이터를 S3 등 값싼 오브젝트 스토리지에 저장해 사실상 무제한 보관
    - 100% 호환: Prometheus의 remote_write를 그대로 받고 PromQL도 그대로 동작 → Grafana 입장에서는 datasource 주소만 바뀜

## 4. 참고
1. Prometheus는 수집기 편에도 나왔는데, DB이기도 하다?
- Prometheus는 수집(Pull)과 저장(TSDB)을 한 몸에 갖춘 도구
- Alloy를 도입하면 수집은 Alloy가 대신하고, Prometheus는 remote_write를 받아주는 **저장소 역할만** 남는다 (`--web.enable-remote-write-receiver` 필요)

2. Loki vs Elasticsearch (로그 DB면 ES 쓰면 되잖아?)

|구분|Elasticsearch|Loki|
|---|---|---|
|인덱싱|로그 본문 전체를 역인덱스로 구축|label(메타데이터)만 인덱싱, 본문은 압축 저장|
|저장 비용|인덱스가 원본보다 커지기도 함 → 비쌈|인덱스가 극도로 작음 → 저렴|
|검색|아무 키워드나 전문 검색이 빠름|label로 범위를 좁힌 뒤 본문은 grep|
|어울리는 곳|로그가 곧 서비스인 검색 플랫폼|메트릭으로 감지하고 로그로 원인을 파는 관측성 파이프라인|

3. 왜 Grafana의 DB들(Loki, Tempo, Mimir)은 전부 Object Storage를 쓸까?
- Grafana 저장소들의 공통 설계 철학: "인덱스는 최소화하고, 원본은 값싼 곳에 통째로 던져라"
- 로컬 디스크나 전용 스토리지 클러스터 없이 S3/GCS만 있으면 되므로, 보존 기간을 늘리는 비용이 선형적이고 운영 부담이 거의 없음

4. HA(High Availability, 고가용성)가 뭐야?
- 시스템 일부가 죽어도 서비스가 끊기지 않고 계속 동작하도록 만드는 구성
- 보통 같은 역할의 서버를 2대 이상 띄워두고(이중화), 하나가 죽으면 나머지가 즉시 이어받는 방식(failover)으로 구현
- Prometheus의 HA가 애매한 이유
  - 클러스터 개념이 없어서, 공식 HA 방법이 그냥 **똑같은 설정의 서버 2대를 나란히 띄우는 것**
  - 2대가 서로를 모르고 각자 따로 수집 → 데이터가 공유되는 게 아니라, 스크랩 시점이 미묘하게 다른 **복사본 2벌**이 생김
  - 쿼리를 어느 쪽에 보내느냐에 따라 값이 살짝 다르고, 한 대가 죽었다 살아나면 그 기간의 데이터는 그 서버에만 구멍이 남
- 반면 Mimir는 여러 노드가 하나의 저장소처럼 동작하는 진짜 분산 시스템이라, 노드 하나가 죽어도 유실 없이 이어받음
