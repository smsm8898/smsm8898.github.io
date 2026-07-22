# public/

## 용도

`hugo` 빌드의 **최종 결과물** — 배포되는 정적 사이트 전체가 여기 생성된다.

## 언제 쓰게 되나

직접 만질 일 없음. 로컬에서 빌드 결과를 확인할 때 들여다보는 정도
(`public/posts/<slug>/index.html`이 생겼는지 등).

**커밋하지 않는다** (`.gitignore` 포함, Hugo 공식 문서 권고). 배포는 push하면
GitHub Actions가 CI에서 새로 빌드해 Pages로 올리는 구조라, 로컬 빌드 결과물이
repo에 있을 이유가 없다. 오래된 로컬 빌드가 남아 있으면 `hugo --gc`나
`rm -rf public/`으로 정리해도 된다.
