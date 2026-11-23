# venaCode 기술 블로그

https://venacode.github.io

제조(MES) 도메인 백엔드 개발자의 기록입니다. [Jekyll Chirpy 테마](https://github.com/cotes2020/jekyll-theme-chirpy)를 사용합니다.

## 글 쓰기

`_posts/YYYY-MM-DD-제목-슬러그.md` 파일을 만들고 아래 front matter를 붙입니다.

```yaml
---
title: "글 제목"
description: >-
  목록·검색에 보이는 한두 문장 요약
date: 2026-11-24 09:00:00 +0900   # 현재 시각보다 미래면 빌드에서 빠짐
categories: [큰분류, 작은분류]      # 최대 2단계
tags: [java, spring]               # 소문자 권장
# mermaid: true                    # mermaid 다이어그램을 쓸 때
# render_with_liquid: false        # 본문에 {{ }}·{% %} 코드가 있을 때
---
```

- 주소는 `/posts/<파일 이름에서 날짜를 뺀 부분>/`이 됩니다.
- 안내 상자: 문단 아래에 `{: .prompt-tip }` / `.prompt-info` / `.prompt-warning` / `.prompt-danger`

## 로컬 미리보기

```bash
bundle install
npm install && npm run build   # 테마 JS 빌드 (처음 한 번, 테마 JS를 고쳤을 때)
bundle exec jekyll serve       # http://127.0.0.1:4000
```

## 배포

`main`에 push하면 `.github/workflows/jekyll.yml`이 빌드해서 GitHub Pages에 올립니다.

## 라이선스

테마는 MIT 라이선스입니다([LICENSE](LICENSE)).
