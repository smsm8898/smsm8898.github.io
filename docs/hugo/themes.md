# themes/

## 용도

테마들의 자리. 이 블로그는 PaperMod를 **git submodule**로 넣었다
(PaperMod 공식 권장 방식).

테마 내부를 열어보면 `layouts/` `assets/` `i18n/` 등 **루트와 같은 구조**가 있는데,
이건 설계다 — union mount가 디렉토리별로 테마 층(기본값)과 루트 층(오버라이드)을
겹치기 때문에, 테마는 그 자체가 Hugo 사이트 모양을 하고 있어야 한다.
`images/`, `theme.toml`, `go.mod`는 Hugo 구조가 아니라 테마 저장소 자체의
메타데이터(README 스크린샷, Hugo Modules 지원)라 빌드와 무관하다.

## 운영 규칙

- **테마 파일을 직접 수정하지 않는다** — submodule이라 수정하면 업데이트가 꼬인다.
  커스텀은 전부 union mount 오버라이드로: 템플릿은 `layouts/`, 스타일은
  `assets/css/extended/`, 문자열은 `i18n/` (각 노트 참고).
- 테마 업데이트:

  ```bash
  git submodule update --remote --merge
  ```

- clone 시 submodule을 같이 받아야 한다 (`--recurse-submodules`).
  빼먹으면 테마가 없어서 빈 화면이 뜬다 — README의 로컬 실행 참고.
