# assets/

## 용도

Hugo Pipes(에셋 파이프라인)로 **처리되는** 전역 리소스 — Sass 컴파일, JS 번들링,
이미지 변환 등을 거쳐 결과물이 사이트에 들어간다. `static/`과의 차이가 이것:
static은 무가공 복사, assets는 가공 대상.

## 언제 쓰게 되나

커스텀 스타일을 넣을 때. PaperMod는 `assets/css/extended/` 밑의 css 파일을
자동으로 테마 스타일 뒤에 이어붙인다 (union mount로 테마의 같은 경로에 합쳐짐):

```
assets/css/extended/custom.css   # 만들기만 하면 적용됨
```

폰트 교체, 코드블록 스타일 조정 같은 작은 커스텀은 전부 이 방식이 정석 —
테마 파일을 직접 고치지 않으므로 테마 업데이트와 충돌하지 않는다.
