# smsm8898.log

기술 공부·리뷰 블로그 — **https://smsm8898.github.io**

공부하거나 직접 써 본 기술을 정리하고 리뷰한다. 주제를 미리 정해두지 않고,
그때그때 다룬 기술을 글로 남긴다.

## 스택

| 항목 | 선택 |
|---|---|
| SSG | [Hugo](https://gohugo.io/) (≥ 0.146) |
| 테마 | [PaperMod](https://github.com/adityatelange/hugo-PaperMod) — git submodule |
| 호스팅 | GitHub Pages — main에 push하면 GitHub Actions가 자동 빌드·배포 |

## 로컬 실행

```bash
git clone --recurse-submodules https://github.com/smsm8898/smsm8898.github.io.git
cd smsm8898.github.io
hugo server -D          # http://localhost:1313 — 저장하면 즉시 반영 (live reload)
```

- `-D` — draft(`draft: true`) 글까지 렌더. 쓰는 중인 글을 보려면 필수 (archetype으로 만든 새 글은 draft로 시작)
- 발행될 모습 그대로 확인하려면 `-D` 없이 `hugo server`
- submodule 없이 클론했다면: `git submodule update --init --recursive`

## 글 쓰기

```bash
hugo new content posts/<slug>/index.md               # 일반 글 (archetypes/posts.md)
hugo new content -k overview posts/<slug>/index.md    # 시리즈 개요 (archetypes/overview.md)
hugo server -D                                        # draft 포함 미리보기
```

- page bundle(디렉토리 + `index.md`) — 스크린샷을 글 옆에 두고 `![](screenshot.png)`로 참조
- 관련 글이 여럿이면 `posts/<주제>/` 하위로 묶어도 된다 (선택 — 정해진 분류 체계는 없음)
- 일반 글 구조: 사용 예시 → 문제 → 정의 → 비교 → 구현 → 참고 (안 맞는 섹션은 삭제)
- 시리즈 개요 구조: 왜 필요한가 → 전체 그림 → 조각별 역할 → 시리즈 목차 → 참고
- 시리즈로 묶을 땐 개요·세부 글 모두 같은 `series: [이름]`을 준다

front matter 규칙:

- `tags` — 다루는 기술·기법 이름 (`Hacker News`, `Airflow`, `Prometheus`, …). 자유롭게
- `categories` — (선택) 넓게 묶고 싶을 때만. 정해진 목록 없음, 안 쓰면 비워둔다
- `date`가 미래 시각이면 그때까지 발행되지 않는다, `draft: true`는 지워야 발행된다

```yaml
---
title: "Hacker News Ranking — 인기 지면에 시간을 넣는 법"
date: 2026-07-23T08:00:00+09:00
tags: [Hacker News, Ranking]
summary: "한 줄 요약"
---
```

## 구조

```
archetypes/posts.md            # 일반 글 템플릿
archetypes/overview.md         # 시리즈 개요 템플릿 (-k overview)
content/
  posts/<slug>/                # 글 — page bundle (index.md + 이미지)
  archives.md                  # /archives/
  search.md                    # /search/ (Fuse.js)
hugo.yaml                      # 사이트 설정
.github/workflows/hugo.yaml    # Pages 배포 workflow
themes/PaperMod/               # 테마 (submodule)
```

## 배포

main에 push → GitHub Actions 빌드 → Pages 배포 (1~2분). `public/`은 커밋하지 않는다.
