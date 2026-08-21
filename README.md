# KAYFABE 2.0 개발 결과 보고서

다중 에이전트 경기 예측·검증 플랫폼(KayFabe)의 개발 결과 보고서 사이트입니다.
Jekyll 로 빌드하며, `main` 푸시 시 GitHub Actions 가 GitHub Pages 에 배포합니다.

## 로컬 실행

```bash
bundle install
bundle exec jekyll serve
```

## 구성

| 경로 | 내용 |
|------|------|
| `index.md` | 표지 — 사업명·팀·기간 등 메타는 front matter 에 있다 |
| `toc.md` | 목차 |
| `0*.md` | 장별 본문. front matter 의 `chapter` 값이 장 순서와 이동 링크를 만든다 |
| `_layouts/` | `cover` (표지) · `report` (본문) |
| `assets/main.scss` | minima 위에 얹는 보고서 스타일 |
