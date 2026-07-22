# layouts/

## 용도

콘텐츠를 HTML로 변환하는 **템플릿**. union mount의 대표 사례 —
테마와 같은 경로에 파일을 두면 **테마 템플릿 대신 내 것이 쓰인다.**

## 언제 쓰게 되나

PaperMod 화면 일부를 고치고 싶을 때. 테마 파일을 직접 수정하는 게 아니라
(submodule이라 수정하면 안 됨), 같은 경로를 프로젝트에 복사해 고친다:

```
themes/PaperMod/layouts/partials/footer.html   # 원본 (건드리지 않음)
layouts/partials/footer.html                   # 복사 후 수정 → 이게 적용됨
```

자주 쓰는 오버라이드 지점:

- `layouts/partials/footer.html` — 푸터 문구
- `layouts/partials/extend_head.html` — `<head>`에 추가 (analytics, 폰트, 댓글 스크립트)
- `layouts/partials/comments.html` — 댓글 시스템 (giscus 등) 붙일 때

주의: 오버라이드한 파일은 테마 업데이트를 자동으로 못 받는다. 최소한만 가져올 것.
