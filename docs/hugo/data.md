# data/

## 용도

템플릿에서 `site.Data.<파일명>`으로 읽을 수 있는 **구조화 데이터**
(JSON, YAML, TOML, XML). 콘텐츠도 설정도 아닌 "표시용 데이터"의 자리.

## 언제 쓰게 되나

같은 데이터를 여러 곳에 뿌리거나, 목록을 콘텐츠와 분리해 관리하고 싶을 때.
예: 읽은 책 목록, 시리즈 정의, 외부 프로젝트 링크 모음.

```yaml
# data/projects.yaml
- name: reco
  url: https://github.com/smsm8898/reco
  desc: 추천 서빙 API
```

커스텀 템플릿(layouts/)에서 `range site.Data.projects`로 렌더링한다.
템플릿 커스텀을 시작하기 전에는 쓸 일이 없다.
