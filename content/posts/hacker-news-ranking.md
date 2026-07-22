---
title: "Hacker News Ranking — 인기 지면에 시간을 넣는 법"
date: 2026-07-23T08:00:00+09:00
categories: [추천시스템]
tags: [hacker-news-ranking, ranking]
summary: "누적 점수 정렬의 문제를 시간 감쇠로 푸는 Hacker News ranking 정리. 공식, gravity의 의미, 상품 인기 지면에 적용할 때의 설계 결정."
---

## 누적 인기 정렬의 문제

인기 지면을 만드는 가장 단순한 방법은 조회수나 판매량을 그대로 내림차순 정렬하는 것이다.
이 방식의 문제는 **한번 1등이 영원한 1등**이라는 것 — 누적치는 시간이 지날수록 커지기만
하므로, 출시된 지 오래된 항목이 상위를 고정 점유하고 새로 뜨는 항목은 올라올 수 없다.
"인기"라는 말에 사람들이 기대하는 것은 사실 "**요즘** 인기"인데, 누적 정렬은 "역대 인기"를
보여준다.

## Hacker News의 답: 시간으로 나눈다

[Hacker News](https://news.ycombinator.com/)의 프론트페이지 랭킹은 이 문제를 공식 하나로 푼다.

```
score = points / (age + 2)^gravity
```

- `points` — 항목이 받은 투표 수 (인기 신호)
- `age` — 게시 후 경과 시간 (HN은 시간(hour) 단위)
- `gravity` — 감쇠 강도, HN 기본값 **1.8**

분자가 인기, 분모가 시간이다. 시간이 지날수록 분모가 거듭제곱으로 커지므로,
점수가 같다면 새 항목이 위로 온다. 오래된 항목이 상위에 남으려면
분모가 커지는 속도를 이길 만큼 분자(투표)를 계속 벌어야 한다.

### gravity — 유일한 튜닝 손잡이

`gravity`가 클수록 시간 페널티가 가팔라진다. 극단을 생각하면 감이 온다:

- `gravity = 0` → 분모가 상수. 누적 인기 정렬과 같아진다.
- `gravity → ∞` → 최신순 정렬과 같아진다.

즉 이 공식은 **"인기순"과 "최신순" 사이를 gravity 하나로 보간**하는 장치다.
1.8이라는 기본값은 HN이 뉴스 피드에 맞게 잡은 지점이고, 지면 성격에 따라 조절하면 된다.

### +2는 왜 붙는가

`age + 2`의 상수 2는 두 가지 일을 한다. `age = 0`일 때 0으로 나누는 것을 막고,
갓 게시된 항목들끼리의 점수 변동을 완만하게 만든다. 상수가 없으면 age가 0에 가까운
구간에서 분모가 급변해서, 게시 직후 몇 분 차이가 순위를 심하게 흔든다.

## 숫자로 감 잡기

gravity 1.8, age를 일(day) 단위로 했을 때:

| 항목 | points | age (일) | score = points/(age+2)^1.8 |
|---|---|---|---|
| 오래된 베스트셀러 | 300 | 6 | 300/42.2 ≈ **7.1** |
| 꾸준한 스테디셀러 | 100 | 3 | 100/18.1 ≈ **5.5** |
| 방금 뜨는 신상 | 40 | 0.5 | 40/5.2 ≈ **7.7** |

누적으로는 300 vs 40으로 압도적인 차이지만, 시간 감쇠 후에는 신상이 베스트셀러를
근소하게 이긴다. 이것이 의도된 동작이다 — 신상은 기회를 얻고, 베스트셀러는
계속 팔리는 한(points가 계속 오르는 한) 다시 올라온다.

## 상품 인기 지면에 적용하기

이 공식을 커머스 인기 지면에 가져올 때 결정할 것이 세 가지 있었다.
([reco](https://github.com/smsm8898/reco) 프로젝트의 인기 상품 API에 적용한 내용이다.)

**1. points를 무엇으로 할 것인가.** HN의 투표 대신 상품 행동 신호(조회·구매)를 쓴다.
스케일이 다른 두 신호(조회는 수백, 구매는 한 자릿수)는 그대로 더하면 조회가 지배하므로,
각각 랭킹을 만든 뒤 [RRF](https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf)로
융합한 점수를 points로 썼다.

**2. age의 단위.** HN은 시간 단위지만, 주간 인기 지면이라면 일(day) 단위가 맞다.
단위를 바꾸는 것은 gravity를 바꾸는 것과 같은 효과 — 같은 gravity라도 단위가 크면
감쇠가 느려진다. "이 지면에서 인기가 며칠짜리 개념인가"에 맞춰 고르면 된다.

**3. 기준 시각의 결정성.** `age = now() - last_event_at`으로 하면 호출 시각마다 결과가
달라진다. 고정 데이터셋에서 테스트를 재현 가능하게 만들려면 `now()` 대신 **풀 안에서
가장 최근 이벤트 시각**을 기준으로 쓸 수 있다 — 라이브 서비스라면 실시간 `now()`가
들어갈 자리다. 같은 이유로 동점일 때는 id로 tie-break해서 정렬을 결정적으로 만든다.

```python
HN_GRAVITY = 1.8
AGE_UNIT_SECONDS = 86400  # day

def hacker_news_rank(rows, gravity=HN_GRAVITY):
    """rows: [{id, score, last_event_at}] — score가 points 역할."""
    now = max(row["last_event_at"] for row in rows)  # 결정성: now() 대신 풀 최신값
    ranked = []
    for row in rows:
        age = (now - row["last_event_at"]).total_seconds() / AGE_UNIT_SECONDS
        hn_score = row["score"] / (age + 2) ** gravity
        ranked.append({**row, "hn_score": hn_score})
    ranked.sort(key=lambda r: (-r["hn_score"], r["id"]))  # 동점은 id로 결정적 tie-break
    return ranked
```

## 장단점과 대안

**좋은 점** — 공식 하나, 튜닝 손잡이 하나(gravity). 결과를 설명하기 쉽다
("점수를 시간으로 나눴다"). 학습이 필요 없어서 콜드스타트가 없다.

**한계** — gravity가 지면 성격에 맞는지는 데이터를 보며 감으로 잡아야 한다.
또 points의 절대량에 민감해서, 트래픽이 적은 카테고리에서는 소수의 이벤트로
순위가 크게 출렁인다.

**대안들** — Reddit의 hot ranking은 points에 log를 씌워 초기 투표에 가중치를 준다
(첫 10표가 다음 100표와 같은 무게). 평점 기반 정렬이라면 표본 크기를 반영하는
[Wilson score interval](https://www.evanmiller.org/how-not-to-sort-by-average-rating.html)이
정석. 시간 감쇠 자체를 지수함수로 하는 변형(exponential decay)도 흔하다.
공통 주제는 하나다 — **원시 누적치를 그대로 정렬하지 말 것.**

## 참고

- [How Hacker News ranking algorithm works](https://medium.com/hacking-and-gonzo/how-hacker-news-ranking-algorithm-works-1d9b0cf2c08d)
- [reco — 추천 서빙 API 학습 프로젝트](https://github.com/smsm8898/reco)
- [Reciprocal Rank Fusion (Cormack et al. 2009)](https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf)
