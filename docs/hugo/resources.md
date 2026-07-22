# resources/

## 용도

Hugo 에셋 파이프라인의 **캐시** (자동 생성). Sass 컴파일 결과, 처리된 이미지
등이 쌓여서 다음 빌드를 빠르게 한다.

## 언제 쓰게 되나

직접 만질 일 없음. 지워도 다음 빌드 때 재생성된다.
`.gitignore`에 포함 — 커밋하지 않는다.

CI에서는 `.github/workflows/hugo.yaml`의 cache restore/save 단계가
같은 역할(빌드 간 캐시)을 actions/cache로 수행한다
(`hugo.yaml`의 `caches.images.dir: :cacheDir/images` 설정과 짝).
