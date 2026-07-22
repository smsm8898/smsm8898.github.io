# static/

## 용도

빌드 시 **가공 없이 `public/` 루트로 그대로 복사**되는 파일.
`static/favicon.ico` → `https://smsm8898.github.io/favicon.ico`.

## 언제 쓰게 되나

- **favicon** — `static/favicon.ico` 등을 두고 `hugo.yaml`의
  `params.assets.favicon`으로 경로 지정 (PaperMod 방식)
- 검증 파일 (Google Search Console의 `googlexxxx.html` 등)
- 글과 무관한 전역 이미지 (프로필 사진, OG 기본 이미지)

글에 들어가는 이미지는 여기 말고 **page bundle**(content 노트 참고)이 낫다 —
글과 이미지가 따로 놀면 나중에 관리가 어렵다.
