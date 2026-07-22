# Hugo + PaperMod 공부 블로그 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Hugo + PaperMod 공부 블로그를 `https://smsm8898.github.io`에 배포하고 첫 글(Hacker News ranking)을 발행한다.

**Architecture:** Hugo 정적 사이트 + PaperMod 테마(git submodule). push하면 GitHub Actions가 빌드해서 GitHub Pages로 배포. 글은 `content/posts/*.md`.

**Tech Stack:** Hugo (brew, ≥ 0.146.0 — PaperMod 최소 요구), PaperMod, GitHub Pages + Actions.

## Global Constraints

- Hugo ≥ **0.146.0** (PaperMod 요구사항). 로컬은 brew 최신, CI는 workflow에 고정된 버전.
- 테마는 git submodule: `git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod` (PaperMod 공식 권장).
- 설정 파일은 `hugo.yaml` (yaml 포맷).
- `baseURL: https://smsm8898.github.io/` — repo 이름은 `smsm8898.github.io` (개인 계정, public).
- `public/`, `resources/`, `.hugo_build.lock`은 커밋하지 않는다 (Hugo 공식 문서: publishDir 커밋 금지).
- 커밋 메시지에 Co-Authored-By trailer 금지 (사용자 규칙).
- **push는 Task 5에서 한 번만** — Task 1~4는 로컬 커밋만 쌓는다 (사용자 규칙).
- 글·문서는 한국어, 회사·실무 경험 언급 금지 (상위 ROADMAP 기밀 경계).

---

### Task 1: Hugo 설치 + 사이트 뼈대 + PaperMod 테마

**Files:**
- Create: `blog/hugo.yaml` (hugo new site가 생성, Task 2에서 내용 교체)
- Create: `blog/.gitignore`
- Create: `blog/themes/PaperMod` (submodule), `blog/.gitmodules`

**Interfaces:**
- Consumes: 없음 (blog/ git repo는 이미 init됨, ROADMAP.md 커밋 존재)
- Produces: Hugo 사이트 디렉토리 구조 (`content/`, `themes/PaperMod/`, `hugo.yaml`) — Task 2~4가 이 위에서 작업

- [ ] **Step 1: Hugo 설치**

```bash
brew install hugo
hugo version
```

Expected: `hugo v0.1xx.x ...` — 0.146.0 이상인지 확인.

- [ ] **Step 2: 사이트 뼈대 생성**

```bash
cd /Users/seungminjang/Desktop/personal/blog
hugo new site . --force --format yaml
```

`--force`: 디렉토리가 비어있지 않아서(ROADMAP.md, .git) 필요. `--format yaml`: hugo.yaml 생성.
Expected: "Congratulations! Your new Hugo site was created" 메시지, `content/ layouts/ static/ archetypes/ hugo.yaml` 생성.

- [ ] **Step 3: .gitignore 작성**

```gitignore
public/
resources/
.hugo_build.lock
```

- [ ] **Step 4: PaperMod submodule 추가**

```bash
cd /Users/seungminjang/Desktop/personal/blog
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

Expected: `.gitmodules` 생성, `themes/PaperMod/`에 테마 클론됨. `ls themes/PaperMod/layouts`로 확인.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: Hugo 사이트 뼈대 + PaperMod 테마 (submodule)"
```

### Task 2: 사이트 설정 + 검색·아카이브 페이지 + 로컬 확인

**Files:**
- Modify: `blog/hugo.yaml` (전체 교체)
- Create: `blog/content/archives.md`
- Create: `blog/content/search.md`

**Interfaces:**
- Consumes: Task 1의 사이트 구조와 PaperMod 테마
- Produces: 동작하는 사이트 설정 — `outputs.home`에 JSON(검색 인덱스), 메뉴 4개(Archives/Categories/Tags/Search). Task 4 글의 categories/tags가 이 메뉴에서 보인다.

- [ ] **Step 1: hugo.yaml 전체 교체**

```yaml
baseURL: https://smsm8898.github.io/
languageCode: ko-kr
defaultContentLanguage: ko
title: seungmin.log
theme: [PaperMod]

enableRobotsTXT: true

pagination:
  pagerSize: 10

minify:
  disableXML: true

# Fuse.js 검색 인덱스 (PaperMod 공식 문서의 검색 설정)
outputs:
  home: [HTML, RSS, JSON]

# CI 캐시 경로 (Hugo 공식 Pages workflow와 짝)
caches:
  images:
    dir: :cacheDir/images

params:
  defaultTheme: auto
  ShowReadingTime: true
  ShowToc: true
  ShowBreadCrumbs: true
  ShowPostNavLinks: true
  ShowCodeCopyButtons: true
  homeInfoParams:
    Title: seungmin.log
    Content: 추천시스템 · 데이터 파이프라인 · 인프라 · 관측성을 공부하고 정리합니다.

menu:
  main:
    - identifier: archives
      name: Archives
      url: /archives/
      weight: 10
    - identifier: categories
      name: Categories
      url: /categories/
      weight: 20
    - identifier: tags
      name: Tags
      url: /tags/
      weight: 30
    - identifier: search
      name: Search
      url: /search/
      weight: 40
```

- [ ] **Step 2: 아카이브·검색 페이지 생성**

`content/archives.md`:

```yaml
---
title: Archives
layout: archives
url: /archives/
summary: archives
---
```

`content/search.md`:

```yaml
---
title: Search
layout: search
url: /search/
summary: search
placeholder: 검색어를 입력하세요
---
```

- [ ] **Step 3: 빌드 확인**

```bash
cd /Users/seungminjang/Desktop/personal/blog
hugo --gc --minify
grep -o "<title>[^<]*" public/index.html
```

Expected: 빌드 에러 없이 `Total in ... ms`, title에 `seungmin.log`.

- [ ] **Step 4: 로컬 미리보기 확인 (ROADMAP Phase 1 완료 기준)**

```bash
hugo server --port 1313 &
sleep 2
curl -s http://localhost:1313/ | grep -o "<title>[^<]*"
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:1313/search/
kill %1
```

Expected: title 출력 + search 페이지 200.

- [ ] **Step 5: Commit**

```bash
git add hugo.yaml content/
git commit -m "feat: 사이트 설정 (한국어·검색·메뉴) + 아카이브·검색 페이지"
```

### Task 3: GitHub Pages 배포 workflow

**Files:**
- Create: `blog/.github/workflows/hugo.yaml`

**Interfaces:**
- Consumes: Task 1~2의 사이트 (submodule 포함 — workflow의 `submodules: recursive`가 PaperMod를 가져온다)
- Produces: push 시 자동 빌드·배포 파이프라인. Task 5가 push 후 이 workflow의 성공을 확인한다.

- [ ] **Step 1: workflow 작성**

Hugo 공식 문서(host-on-github-pages)의 권장 workflow를 그대로 사용, `TZ`만 `Asia/Seoul`로 변경:

```yaml
name: Build and deploy
on:
  push:
    branches:
      - main
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write
concurrency:
  group: pages
  cancel-in-progress: false
defaults:
  run:
    shell: bash
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      DART_SASS_VERSION: 1.101.0
      GO_VERSION: 1.26.4
      HUGO_VERSION: 0.164.0
      NODE_VERSION: 24.18.0
      TZ: Asia/Seoul
    steps:
      - name: Checkout
        uses: actions/checkout@v7
        with:
          submodules: recursive
          fetch-depth: 0
          lfs: false
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v6
      - name: Create a local tools directory
        run: mkdir -p "${HOME}/.local"
      - name: Install Go
        if: hashFiles('go.mod') != ''
        uses: actions/setup-go@v6
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: false
      - name: Install Node.js
        if: hashFiles('package-lock.json') != ''
        uses: actions/setup-node@v6
        with:
          node-version: ${{ env.NODE_VERSION }}
      - name: Install Dart Sass
        run: |
          echo "Installing Dart Sass ${DART_SASS_VERSION}..."
          curl -sfL --output-dir "${{ runner.temp }}" -O "https://github.com/sass/dart-sass/releases/download/${DART_SASS_VERSION}/dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          tar -C "${HOME}/.local" -xf "${{ runner.temp }}/dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          echo "${HOME}/.local/dart-sass" >> "${GITHUB_PATH}"
      - name: Install Hugo
        run: |
          echo "Installing Hugo ${HUGO_VERSION}..."
          curl -sfL --output-dir "${{ runner.temp }}" -O "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_linux-amd64.tar.gz"
          mkdir "${HOME}/.local/hugo"
          tar -C "${HOME}/.local/hugo" -xf "${{ runner.temp }}/hugo_${HUGO_VERSION}_linux-amd64.tar.gz"
          echo "${HOME}/.local/hugo" >> "${GITHUB_PATH}"
      - name: Log tool versions
        run: |
          echo "Logging tool versions..."
          command -v sass &> /dev/null && echo "Dart Sass: $(sass --version)" || echo "Dart Sass: not installed"
          command -v go &> /dev/null && echo "Go: $(go version)" || echo "Go: not installed"
          command -v hugo &> /dev/null && echo "Hugo: $(hugo version)" || echo "Hugo: not installed"
          command -v node &> /dev/null && echo "Node.js: $(node --version)" || echo "Node.js: not installed"
      - name: Configure Git
        run: |
          echo "Configuring Git..."
          git config --global core.quotepath false
      - name: Fetch full Git history
        run: |
          if [[ $(git rev-parse --is-shallow-repository) == true ]]; then
            echo "Fetching full Git history..."
            git fetch --unshallow
          fi
      - name: Initialize Git submodules
        run: |
          if [[ -f .gitmodules ]]; then
            echo "Initializing Git submodules..."
            git submodule update --init --recursive
          fi
      - name: Install Node.js dependencies
        run: |
          if [[ -f package-lock.json ]]; then
            echo "Installing Node.js dependencies..."
            npm ci
          fi
      - name: Cache restore
        id: cache-restore
        uses: actions/cache/restore@v6
        with:
          path: ${{ runner.temp }}/.cache/hugo
          key: hugo-${{ github.run_id }}
          restore-keys: hugo-
      - name: Build
        run: |
          echo "Building the project..."
          hugo build \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/" \
            --cacheDir "${{ runner.temp }}/.cache/hugo"
      - name: Cache save
        uses: actions/cache/save@v6
        with:
          path: ${{ runner.temp }}/.cache/hugo
          key: ${{ steps.cache-restore.outputs.cache-primary-key }}
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v5
        with:
          include-hidden-files: false
          path: ./public
  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v5
```

- [ ] **Step 2: yaml 문법 확인**

```bash
cd /Users/seungminjang/Desktop/personal/blog
python3 -c "import yaml, sys; yaml.safe_load(open('.github/workflows/hugo.yaml')); print('OK')"
```

Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add .github/
git commit -m "ci: GitHub Pages 배포 workflow (Hugo 공식 권장)"
```

### Task 4: 첫 글 — Hacker News Ranking

**Files:**
- Create: `blog/content/posts/hacker-news-ranking.md`

**Interfaces:**
- Consumes: Task 2의 설정 (categories/tags 메뉴, 검색 인덱스)
- Produces: 첫 발행 글 — category `추천시스템`, tags `hacker-news-ranking`, `ranking` (ROADMAP의 카테고리/태그 체계 검증)

- [ ] **Step 1: 글 작성**

`content/posts/hacker-news-ranking.md` 전문:

````markdown
---
title: "Hacker News Ranking — 인기 지면에 시간을 넣는 법"
date: 2026-07-23T12:00:00+09:00
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
````

- [ ] **Step 2: 빌드로 글 렌더 확인**

```bash
cd /Users/seungminjang/Desktop/personal/blog
hugo --gc --minify
ls public/posts/hacker-news-ranking/index.html
ls public/categories/추천시스템/index.html
ls public/tags/hacker-news-ranking/index.html
```

Expected: 세 파일 모두 존재 (글 + 카테고리 + 태그 페이지 생성 확인).

- [ ] **Step 3: Commit**

```bash
git add content/posts/
git commit -m "post: Hacker News ranking — 인기 지면의 시간 감쇠"
```

### Task 5: GitHub repo 생성 + push + Pages 설정 + 배포 확인

**Files:**
- Modify: `blog/ROADMAP.md` (진행 상황 갱신)

**Interfaces:**
- Consumes: Task 1~4의 로컬 커밋 전부, Task 3의 workflow
- Produces: 살아있는 블로그 `https://smsm8898.github.io` (ROADMAP Phase 2·3 완료 기준)

- [ ] **Step 1: repo 생성 (개인 계정, push 없이)**

```bash
gh repo create smsm8898.github.io --public --source /Users/seungminjang/Desktop/personal/blog --remote origin --description "기술 공부 블로그 — 추천시스템·데이터 파이프라인·인프라·관측성"
```

Expected: `https://github.com/smsm8898/smsm8898.github.io` 생성, origin remote 추가됨. (gh는 이미 smsm8898 계정으로 로그인 확인됨)

- [ ] **Step 2: Pages source를 GitHub Actions로 설정**

```bash
gh api -X POST repos/smsm8898/smsm8898.github.io/pages -f build_type=workflow
```

Expected: 201. 이미 Pages가 존재해 409가 나오면:

```bash
gh api -X PUT repos/smsm8898/smsm8898.github.io/pages -f build_type=workflow
```

- [ ] **Step 3: push (전체에서 유일한 push)**

```bash
cd /Users/seungminjang/Desktop/personal/blog
git push -u origin main
```

- [ ] **Step 4: workflow 성공 확인**

```bash
sleep 10
gh run list --repo smsm8898/smsm8898.github.io --limit 1
gh run watch --repo smsm8898/smsm8898.github.io --exit-status $(gh run list --repo smsm8898/smsm8898.github.io --limit 1 --json databaseId --jq '.[0].databaseId')
```

Expected: `Build and deploy` run이 `completed / success`.

- [ ] **Step 5: 라이브 확인 (ROADMAP Phase 2·3 완료 기준)**

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://smsm8898.github.io/
curl -s https://smsm8898.github.io/ | grep -o "<title>[^<]*"
curl -s -o /dev/null -w "%{http_code}\n" https://smsm8898.github.io/posts/hacker-news-ranking/
```

Expected: 200 / `seungmin.log` / 200. (Pages 첫 배포는 전파에 1~2분 걸릴 수 있음 — 404면 잠시 후 재시도)

- [ ] **Step 6: ROADMAP 진행 상황 갱신 + 마무리 커밋·push**

`blog/ROADMAP.md`의 `## 목표` 위에 추가:

```markdown
## 진행 상황 (2026-07-23)

- **Phase 1~3 완료.** `https://smsm8898.github.io` 라이브, 첫 글(Hacker News ranking) 발행.
```

```bash
git add ROADMAP.md
git commit -m "docs: Phase 1~3 완료 기록"
git push
```
