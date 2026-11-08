# 햄의 AI & 개발 블로그

AI 공부와 자동화 기록을 남기는 개인 블로그입니다.
[Jekyll](https://jekyllrb.com/) + [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 테마로 만들었고, GitHub Pages로 배포됩니다.

- 사이트: https://hm-zy.github.io
- 배포: `main` 브랜치에 푸시하면 GitHub Actions가 빌드하고 Pages로 배포합니다 (`.github/workflows/pages-deploy.yml`).

## 글 쓰는 법

1. `_posts/` 폴더에 `YYYY-MM-DD-제목.md` 형식으로 파일을 만듭니다.
2. 맨 위에 머리말을 적고 본문을 마크다운으로 씁니다.

```markdown
---
title: 글 제목
date: 2026-11-08 20:00:00 +0900
categories: [Blog]
tags: [AI, Automation]
---

본문...
```

3. `main` 브랜치에 커밋하고 푸시하면 몇 분 뒤 사이트에 반영됩니다.

## 폴더 구조

| 경로 | 설명 |
| --- | --- |
| `_posts/` | 블로그 글 (마크다운) |
| `_tabs/` | 상단 메뉴 페이지 (About, Archives, Categories, Tags) |
| `_data/contact.yml` | 사이드바 연락처 링크 |
| `_data/share.yml` | 글 하단 공유 버튼 |
| `_config.yml` | 사이트 설정 (제목, 주소, 언어, 시간대) |
| `.github/workflows/pages-deploy.yml` | 빌드·배포 워크플로 |

## 설정 메모

- `lang: ko-KR` — 테마에 들어 있는 한국어 로케일을 씁니다.
- `timezone: Asia/Seoul` — 글 작성 시각과 목록의 날짜가 한국 시간 기준입니다.
- 세부 옵션은 [Chirpy 문서](https://chirpy.cotes.page/)를 참고하세요.

## 로컬에서 미리 보기 (선택)

Ruby 3.1 이상이 필요합니다.

```bash
bundle install
bundle exec jekyll s
```
