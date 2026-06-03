---
name: html-generation
description: "디자인 설계서를 단일 HTML 파일로 변환하는 규칙과 코드 패턴. 시맨틱 마크업, CSS 구현, 반응형, 접근성 기준을 정의합니다. HTML코딩, CSS구현, 웹변환 시 사용됩니다."
---

# HTML 생성 규칙 및 코드 패턴

## 개요

Designer의 설계서를 단일 `.html` 파일로 변환할 때 준수해야 하는 기술 규격, 코드 패턴, 품질 기준을 정의합니다.

---

## 파일 구조 템플릿

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="description" content="자동차 B2B 영업 시장 분석 리포트 - {날짜}">
    <title>B2B Auto Market Report - {YYYY-MM-DD}</title>
    <style>
        /* === RESET === */
        /* === DESIGN TOKENS === */
        /* === LAYOUT === */
        /* === COMPONENTS === */
        /* === RESPONSIVE === */
        /* === PRINT === */
    </style>
</head>
<body>
    <header class="report-header">...</header>
    <main class="report-main">
        <section class="hero-section" aria-label="핵심 인사이트">...</section>
        <section class="stats-section" aria-label="핵심 지표">...</section>
        <section class="items-section" aria-label="상세 분석">...</section>
        <section class="coverage-section" aria-label="영역 커버리지">...</section>
    </main>
    <footer class="report-footer">...</footer>
    <!-- Optional: Minimal JS for interactions -->
    <script>...</script>
</body>
</html>
```

---

## CSS 구현 패턴

### 1. Reset (필수)

```css
*, *::before, *::after {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
}

body {
    font-family: var(--font-family);
    font-size: var(--text-base);
    line-height: 1.6;
    color: var(--color-text);
    background: var(--color-bg);
    -webkit-font-smoothing: antialiased;
}

a {
    color: var(--color-primary-600);
    text-decoration: none;
}
a:hover {
    text-decoration: underline;
}
```

### 2. Design Tokens (CSS Custom Properties)

```css
:root {
    /* Colors */
    --color-primary-50: #EFF6FF;
    --color-primary-100: #DBEAFE;
    --color-primary-500: #3B82F6;
    --color-primary-600: #2563EB;
    --color-primary-700: #1D4ED8;

    --color-success: #22C55E;
    --color-warning: #F59E0B;
    --color-danger: #EF4444;

    --color-bg: #F8FAFC;
    --color-card: #FFFFFF;
    --color-text: #1E293B;
    --color-text-secondary: #64748B;
    --color-border: #E2E8F0;

    /* Typography */
    --font-family: system-ui, -apple-system, 'Segoe UI', sans-serif;
    --text-xs: 0.75rem;
    --text-sm: 0.8125rem;
    --text-base: 0.9375rem;
    --text-lg: 1.125rem;
    --text-xl: 1.375rem;
    --text-2xl: 1.75rem;

    /* Spacing */
    --space-1: 4px;
    --space-2: 8px;
    --space-3: 12px;
    --space-4: 16px;
    --space-5: 20px;
    --space-6: 24px;
    --space-8: 32px;
    --space-10: 40px;
    --space-12: 48px;

    /* Radius */
    --radius-sm: 6px;
    --radius-md: 10px;
    --radius-lg: 16px;
    --radius-full: 9999px;

    /* Shadow */
    --shadow-sm: 0 1px 2px rgba(0,0,0,0.05);
    --shadow-md: 0 4px 12px rgba(0,0,0,0.08);
    --shadow-lg: 0 8px 24px rgba(0,0,0,0.12);
}
```

### 3. Layout Patterns

```css
/* Container */
.report-main {
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 var(--space-6);
}

/* Section spacing */
.report-main > section {
    margin-bottom: var(--space-12);
}

/* Grid - Hero (3 columns) */
.hero-cards {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: var(--space-6);
}

/* Grid - Items (2 columns) */
.item-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: var(--space-6);
}

/* Grid - Stats (auto-fit) */
.stats-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
    gap: var(--space-4);
}
```

### 4. Component Patterns

#### Hero Card
```css
.hero-card {
    padding: var(--space-6);
    border-radius: var(--radius-lg);
    color: white;
    display: flex;
    flex-direction: column;
    gap: var(--space-3);
}
.hero-card:nth-child(1) {
    background: linear-gradient(135deg, #1E40AF, #3B82F6);
}
.hero-card:nth-child(2) {
    background: linear-gradient(135deg, #1E3A5F, #60A5FA);
}
.hero-card:nth-child(3) {
    background: linear-gradient(135deg, #334155, #94A3B8);
}
```

#### Item Card
```css
.item-card {
    background: var(--color-card);
    border: 1px solid var(--color-border);
    border-radius: var(--radius-md);
    padding: var(--space-6);
    transition: box-shadow 0.2s ease;
}
.item-card:hover {
    box-shadow: var(--shadow-md);
}
```

#### Reliability Bar
```css
.reliability-bar {
    height: 8px;
    background: var(--color-border);
    border-radius: var(--radius-full);
    overflow: hidden;
}
.reliability-fill {
    height: 100%;
    border-radius: var(--radius-full);
    transition: width 0.5s ease;
}
.reliability-fill.high { background: var(--color-success); }
.reliability-fill.medium { background: var(--color-warning); }
.reliability-fill.low { background: var(--color-danger); }
```

#### Area Tag
```css
.area-tag {
    display: inline-block;
    padding: var(--space-1) var(--space-3);
    border-radius: var(--radius-full);
    font-size: var(--text-xs);
    font-weight: 500;
}
.area-tag.market { background: #DBEAFE; color: #1D4ED8; }
.area-tag.oem { background: #EDE9FE; color: #6D28D9; }
.area-tag.tech { background: #D1FAE5; color: #065F46; }
.area-tag.policy { background: #FEF3C7; color: #92400E; }
.area-tag.competition { background: #FFE4E6; color: #9F1239; }
.area-tag.customer { background: #CFFAFE; color: #155E75; }
```

### 5. Responsive Patterns

```css
@media (max-width: 1023px) {
    .item-grid {
        grid-template-columns: 1fr;
    }
}

@media (max-width: 767px) {
    .hero-cards {
        grid-template-columns: 1fr;
    }
    .report-main {
        padding: 0 var(--space-4);
    }
    :root {
        --text-2xl: 1.5rem;
        --text-xl: 1.25rem;
    }
}
```

### 6. Print Styles

```css
@media print {
    body {
        background: white;
        font-size: 11pt;
    }
    .report-header {
        border-bottom: 2px solid #000;
        padding-bottom: 12px;
    }
    .item-card {
        break-inside: avoid;
        border: 1px solid #ccc;
        box-shadow: none;
    }
    .hero-card {
        background: #f0f0f0 !important;
        color: #000 !important;
        border: 1px solid #999;
    }
    a[href]::after {
        content: " (" attr(href) ")";
        font-size: 9pt;
        color: #666;
    }
    .stats-grid {
        grid-template-columns: repeat(5, 1fr);
    }
}
```

---

## 데이터 바인딩 규칙

리서치 데이터를 HTML에 삽입할 때의 매핑 규칙:

| 데이터 필드 | HTML 위치 | 형식 |
|------------|-----------|------|
| 항목 제목 | `.item-card h3` | 텍스트 그대로 |
| 신뢰도 점수 | `.reliability-fill` width + 라벨 | `{score}%` + `{score}/100` |
| 신뢰도 등급 | `.reliability-fill` 클래스 | high(80+) / medium(60-79) / low(<60) |
| 출처 URL | `.source-link href` | 전체 URL, target="_blank" rel="noopener" |
| 출처명 | `.source-link` 텍스트 | 매체명 |
| 발행일 | `.item-date` | YYYY-MM-DD 형식 |
| 영역 태그 | `.area-tag` 클래스 + 텍스트 | 매핑 테이블 참조 |
| What | `.insight-what` | 1줄 텍스트 |
| So What | `.insight-so-what` | 2~3줄 텍스트 |
| Now What | `.insight-now-what` | 1~2줄 텍스트, Bold |
| 순위 | `.rank-badge` | #1 ~ #10 |

---

## 접근성 체크리스트

- [ ] 모든 섹션에 `aria-label` 또는 시맨틱 heading 존재
- [ ] 이미지 대체 텍스트 (SVG inline 사용 시 `aria-hidden="true"` + 별도 라벨)
- [ ] 링크 텍스트가 의미 있음 ("여기를 클릭" 금지)
- [ ] 색상 대비 4.5:1 이상 (본문), 3:1 이상 (대형 텍스트)
- [ ] 키보드 네비게이션 가능 (focus outline 유지)
- [ ] lang 속성 설정 (`<html lang="ko">`)

---

## 금지 사항

| 금지 항목 | 이유 |
|----------|------|
| 외부 CSS/JS CDN | 단일 파일 완결성 |
| 웹 폰트 (Google Fonts 등) | 외부 의존성 |
| `<img src="http...">` | 외부 이미지 |
| `document.write()` | 보안 + 성능 |
| `eval()` | 보안 |
| `!important` 남용 | 유지보수성 (Print 제외) |
| inline style 속성 | `<style>` 태그로 통합 (동적 값 제외) |

---

## 파일 검증 (셀프 테스트)

구현 완료 후 아래 확인:

1. **구조 검증**: `<!DOCTYPE html>` 시작, 닫는 태그 매칭
2. **데이터 완전성**: 10개 항목 모두 존재, 점수/출처 누락 없음
3. **반응형**: 375px, 768px, 1024px 각각에서 레이아웃 의도대로
4. **인쇄**: `@media print` 적용 시 깨짐 없음
5. **파일 크기**: 500KB 이하
6. **인코딩**: UTF-8 저장 확인
