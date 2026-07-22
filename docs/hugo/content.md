# content/

## 용도

블로그의 본체 — 마크다운 글과 페이지 리소스. 디렉토리 구조가 곧 URL 구조다
(`content/posts/foo.md` → `/posts/foo/`).

현재 구성:

```
content/
  posts/        # 글
  archives.md   # /archives/ — layout: archives 지정만 하는 빈 페이지
  search.md     # /search/  — layout: search (Fuse.js)
```

## 언제 쓰게 되나

항상. 알아두면 좋은 것 하나 — 글에 이미지를 넣고 싶으면 **page bundle**로 바꾼다:

```
content/posts/my-post.md              # 파일 하나 (이미지 없음)
content/posts/my-post/index.md        # 디렉토리 + index.md
content/posts/my-post/diagram.png     # 글에서 ![](diagram.png)로 참조
```

글과 이미지가 한 디렉토리에 살아서 관리가 깔끔하다.
