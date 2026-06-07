---
name: html-generation
description: "디자인 설계서를 단일 HTML 파일로 변환하는 규칙과 코드 패턴. 시맨틱 마크업, CSS 구현, 반응형, 접근성 기준을 정의합니다. HTML코딩, CSS구현, 웹변환 시 사용됩니다."
---

# HTML 생성 규칙

## 기본 구조

- `<!DOCTYPE html>`, `<html lang="ko">`, viewport meta, UTF-8
- 시맨틱: header, main, section, article, footer
- 인라인 CSS만 (외부 의존성 없음)

## CSS 기본

```css
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
body { font-family: system-ui, sans-serif; font-size: 15px; line-height: 1.6; color: #1E293B; background: #F8FAFC; }
.report-main { max-width: 900px; margin: 0 auto; padding: 0 20px; }
@media (max-width: 767px) { .report-main { padding: 0 12px; } h1 { font-size: 20px; } }
```

## 데이터 매핑

| 데이터 | 클래스 | 비고 |
|--------|--------|------|
| 제목 | `.item-card h3` | |
| 점수 | `.score-badge` | 25+초록/18+주황/나머지빨강 |
| 출처 | `<a target="_blank" rel="noopener">` | |
| 카테고리 | `.category-tag` | 카테고리별 색상 |
| What/So What/Now What | `.insight-*` | Now What은 Bold |

## 금지

- 외부 CSS/JS/CDN, 웹폰트, 외부 이미지
- `!important` (print 제외)

## 완료 체크

5개 항목 모두 삽입, 점수색상 매핑, 모바일 가로스크롤 없음, 출처 새탭, 150KB 이하
