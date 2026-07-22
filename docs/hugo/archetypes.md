# archetypes/

## 용도

`hugo new content posts/foo.md` 실행 시 새 파일에 채워지는 **front matter 템플릿**.
기본 생성된 `default.md`가 title·date·draft를 자동으로 채워준다.

## 언제 쓰게 되나

글 템플릿을 통일하고 싶을 때. 이 블로그는 archetype 두 개를 쓴다.

- `archetypes/posts.md` — **일반 글**. 6단 구조(사용 예시 → 문제 → 정의 → 비교 →
  구현 → 참고). 경로 첫 디렉토리가 `posts/`라 자동 선택된다.

  ```bash
  hugo new content posts/<slug>/index.md
  ```

- `archetypes/overview.md` — **시리즈 개요(허브)**. 다른 구조(왜 필요한가 → 전체 그림
  → 조각별 역할 → 시리즈 목차 → 참고)에 `series` 필드까지 채워 시작한다.

  ```bash
  hugo new content -k overview posts/<slug>/index.md
  ```

## archetype 선택 규칙

archetype은 **콘텐츠 경로의 첫 디렉토리(= content type)**로 자동 선택된다
(`posts/...` → `archetypes/posts.md`). 그와 다른 archetype을 쓰려면 `-k/--kind`로
명시한다 — `-k overview`면 경로가 `posts/` 밑이어도 `archetypes/overview.md`가 쓰인다.
([공식 문서 — Archetypes](https://gohugo.io/content-management/archetypes/))

템플릿 문법에서 `.File.ContentBaseName`은 page bundle의 경우 디렉토리 이름(slug)을
가리킨다 — 그래서 `hacker-news-ranking`이 title 초안 "Hacker News Ranking"이 된다.
