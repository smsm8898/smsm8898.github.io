# blog 구현 로드맵

> 상위 계획: `../ROADMAP.md`. 이 문서는 공부 블로그를 완성하기까지의 세부 단계를 정의한다.
> 5단계 재구현 로드맵과는 **독립 트랙** — 단계 번호 없이 언제든 진행 가능.

## 진행 상황 (2026-07-23)

- **Phase 1~3 완료.** `https://smsm8898.github.io` 라이브, 첫 글(Hacker News ranking) 발행.

## 목표

재구현 프로젝트에서 사용한 기술 스택을 주제별 글로 정리하는 **기술 공부 블로그**.
repo `docs/` 학습 노트가 "이 repo에서 왜 이렇게 했나"라면, 블로그 글은 "이 기술 자체가 무엇인가"를 다룬다.

- 예: reco의 popular에 쓴 Hacker News ranking, infra-apps의 Prometheus/Loki, airflow의 DAG·task 개념과 장단점
- 상위 로드맵의 기밀 경계 원칙을 동일하게 적용한다: 회사·실무 경험을 언급하지 않고, 독립 학습 프로젝트의 기술 정리로 표현한다.
- 문서·글은 한국어로 작성한다.

## 스택

| 항목 | 선택 | 비고 |
|---|---|---|
| SSG | Hugo | Go 단일 바이너리 (`brew install hugo`), 의존성 관리 없음 |
| 테마 | PaperMod | git submodule로 설치 (공식 문서 권장 방식) |
| 호스팅 | GitHub Pages | repo 이름 `smsm8898.github.io` → 루트 도메인 서빙 |
| 배포 | GitHub Actions | Hugo 공식 문서의 Pages workflow, push마다 자동 빌드·배포 |

## repo 레이아웃

```
blog/                       # 로컬 디렉토리 이름. GitHub repo는 smsm8898.github.io
  hugo.yaml                 # 사이트 설정 (한국어 기본, 다크모드, 검색, 태그)
  content/
    posts/<slug>.md         # 글 (front matter에 categories/tags)
  themes/PaperMod/          # git submodule
  .github/workflows/        # Pages 배포 workflow
  ROADMAP.md
```

## 콘텐츠 구조

- **categories = 큰 축 4개**: `추천시스템`, `데이터 파이프라인`, `인프라`, `관측성`
- **tags = 세부 기술**: `hacker-news-ranking`, `airflow`, `prometheus`, `loki`, `argocd` 등
- 예: Hacker News ranking 글 → category `추천시스템`, tags `hacker-news-ranking`, `ranking`

## 글쓰기 흐름 (완성 후 일상)

```
1. content/posts/에 마크다운 작성
2. hugo server -D 로 로컬 미리보기
3. 커밋 → push → 1~2분 뒤 블로그 반영
```

## Phase 목록

### Phase 1: Skeleton
- `hugo new site`로 뼈대 생성, PaperMod를 git submodule로 추가
- `hugo.yaml` 설정: `defaultContentLanguage: ko`, baseURL, 다크모드, 검색(Fuse.js), 태그/카테고리 페이지
- **완료 기준**: `hugo server`로 로컬에서 빈 블로그가 뜬다.

### Phase 2: 배포
- GitHub 개인 계정(smsm8898)에 `smsm8898.github.io` public repo 생성
- Hugo 공식 문서의 GitHub Pages workflow 추가, Pages source를 GitHub Actions로 설정
- **완료 기준**: `https://smsm8898.github.io` 접속 시 블로그가 뜬다.

### Phase 3: 첫 글
- Hacker News ranking 알고리즘 정리 글 초안 (category `추천시스템`)
- 카테고리/태그 체계가 실제 글에서 동작하는지 확인
- **완료 기준**: 첫 글이 카테고리/태그와 함께 배포된 블로그에서 보인다.

## 설계 결정 (진행하며 갱신)

- **Hugo + PaperMod 선택** (브레인스토밍에서 결정): "글쓰기에 집중, 세팅 최소" 방향. Jekyll(Chirpy)은 한국 개발 블로그에서 흔하지만 로컬 미리보기에 Ruby+bundler가 필요해 제외 — Hugo는 단일 바이너리로 로컬 미리보기가 즉시 뜬다. Astro 등 직접 구축은 글쓰기 시작이 늦어져 제외.
- **테마는 git submodule**: PaperMod 공식 문서 권장 방식. 테마 업데이트는 submodule pull로 처리.
- **블로그는 독립 트랙**: 5단계 재구현 로드맵의 순서와 무관하게 진행한다. 글감은 각 repo 작업에서 나온다.
- **repo `docs/` 노트와의 관계**: 자동 연동하지 않는다. docs/는 repo 맥락의 의사결정 기록, 블로그는 기술 자체의 정리 — 서로 다른 글이다.
