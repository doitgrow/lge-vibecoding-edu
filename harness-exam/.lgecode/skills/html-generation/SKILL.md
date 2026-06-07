---
name: html-generation
description: "디자인 설계서를 단일 HTML 파일로 변환하는 규칙과 코드 패턴. 시맨틱 마크업, CSS 구현, 반응형, 접근성 기준을 정의합니다. HTML코딩, CSS구현, 웹변환 시 사용됩니다."
---

# HTML 생성 규칙

## 파일 템플릿

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>B2B 시장 분석 리포트 - {날짜}</title>
  <style>
    /* 여기에 모든 CSS */
  </style>
</head>
<body>
  <header class="report-header">제목 + 날짜</header>
  <main class="report-main">
    <section class="hero-section">핵심 인사이트 카드</section>
    <section class="stats-section">KPI 수치</section>
    <section class="items-section">5개 항목 카드</section>
  </main>
  <footer class="report-footer">면책 + 메타</footer>
</body>
</html>
```

---

## 필수 CSS 패턴

```css
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: system-ui, -apple-system, sans-serif;
  font-size: 15px;
  line-height: 1.6;
  color: #1E293B;
  background: #F8FAFC;
}

.report-main {
  max-width: 900px;
  margin: 0 auto;
  padding: 0 20px;
}

.report-main > section {
  margin-bottom: 32px;
}

/* 반응형 (모바일) */
@media (max-width: 767px) {
  .report-main { padding: 0 12px; }
  h1 { font-size: 20px; }
}
```

---

## 데이터 매핑 규칙

| 데이터 | HTML 위치 | 형식 |
|--------|-----------|------|
| 제목 | `.item-card h3` | 텍스트 |
| 점수 | `.score-badge` | {X}/30 |
| 점수색상 | badge 배경색 | 25+초록/18+주황/나머지빨강 |
| 출처URL | `<a>` 태그 | target="_blank" rel="noopener" |
| 영역태그 | `.category-tag` | 카테고리별 색상 |
| What | `.insight-what` | 1줄 |
| So What | `.insight-so-what` | 1~2줄 |
| Now What | `.insight-now-what` | 1줄, Bold |

---

## 금지 사항

- 외부 CSS/JS CDN
- 웹폰트 (Google Fonts 등)
- 외부 이미지 (`<img src="http...">`)
- `!important` 남용 (print 제외)

---

## 완료 전 체크

1. `<!DOCTYPE html>` 존재
2. `<html lang="ko">` 존재
3. 5개 항목 데이터 모두 삽입됨
4. 점수별 색상 올바르게 적용됨
5. 모바일에서 가로 스크롤 없음
6. 출처 링크 새 탭 열림
7. 파일 크기 150KB 이하
