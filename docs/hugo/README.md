# Hugo 디렉토리 구조 노트

`hugo new site`가 만드는 디렉토리들의 용도와 "언제 쓰게 되나" 정리.
근거: [공식 문서 — Directory structure](https://gohugo.io/getting-started/directory-structure/)

## 핵심 개념: union mount

Hugo는 **테마와 프로젝트의 같은 이름 디렉토리를 하나의 가상 파일시스템으로 합친다.**
같은 경로에 파일이 둘 다 있으면 **프로젝트 쪽이 이긴다.**

빈 디렉토리들이 존재하는 이유가 이것 — 전부 "테마를 덮어쓸 수 있는 자리"다.
테마(PaperMod)를 그대로 쓰는 동안은 비어 있는 게 정상이고, git은 빈 디렉토리를
추적하지 않으므로 커밋에도 안 들어간다. 지워도 되고, 필요할 때 다시 만들면 된다.

## 디렉토리 목록

| 디렉토리 | 한 줄 요약 | 작성/생성 |
|---|---|---|
| [archetypes/](archetypes.md) | `hugo new content`의 front matter 템플릿 | 작성 |
| [assets/](assets.md) | 파이프라인 처리되는 리소스 (커스텀 CSS/JS) | 작성 |
| [content/](content.md) | 글과 페이지 | 작성 |
| [data/](data.md) | 템플릿에서 읽는 데이터 파일 | 작성 |
| [i18n/](i18n.md) | 다국어 번역 문자열 | 작성 |
| [layouts/](layouts.md) | 템플릿 — 테마 화면 오버라이드 지점 | 작성 |
| [static/](static.md) | 사이트 루트로 그대로 복사 (favicon 등) | 작성 |
| [themes/](themes.md) | 테마 (PaperMod submodule) | 작성(외부) |
| [resources/](resources.md) | 파이프라인 캐시 — 커밋 금지 | 생성 |
| [public/](public.md) | 빌드 결과물 — 커밋 금지 | 생성 |
