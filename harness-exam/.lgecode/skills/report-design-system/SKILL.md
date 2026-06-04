---
name: report-design-system
description: "비즈니스 리포트의 시각 디자인 시스템. 색상 팔레트, 타이포그래피, 컴포넌트 규격, 레이아웃 패턴을 정의합니다. 리포트디자인, 시각화, UI설계, HTML리포트 시 사용됩니다."
---

# 비즈니스 리포트 디자인 시스템

## 개요

자동차 B2B 시장분석 리포트의 시각적 일관성과 정보 전달력을 보장하는 디자인 시스템입니다.
Designer 에이전트는 이 시스템을 기반으로 레이아웃을 설계하고, HTML-Maker는 이를 충실히 구현합니다.

---

## 디자인 토큰 (Design Tokens)

### 색상 시스템

```
┌─────────────────────────────────────────────────┐
│  Primary Palette (브랜드/구조)                   │
├─────────────────────────────────────────────────┤
│  --color-primary-900: #0F172A  (가장 진한)      │
│  --color-primary-800: #1E293B  (헤더 배경)      │
│  --color-primary-700: #334155  (부제목)         │
│  --color-primary-600: #475569  (본문 강조)      │
│  --color-primary-400: #94A3B8  (캡션)           │
│  --color-primary-200: #E2E8F0  (보더)           │
│  --color-primary-100: #F1F5F9  (카드 배경)      │
│  --color-primary-50:  #F8FAFC  (페이지 배경)    │
├─────────────────────────────────────────────────┤
│  Semantic Palette (의미/상태)                    │
├─────────────────────────────────────────────────┤
│  --color-success:  #22C55E  (기회, 긍정, 높은 점수) │
│  --color-warning:  #F59E0B  (주의, 중간 점수)   │
│  --color-danger:   #EF4444  (위험, 낮은 점수)   │
│  --color-info:     #3B82F6  (정보, 링크)        │
│  --color-accent:   #8B5CF6  (강조, 하이라이트)  │
├─────────────────────────────────────────────────┤
│  Category Palette (영역 구분)                    │
├─────────────────────────────────────────────────┤
│  시장 동향:  #3B82F6 (Blue)                     │
│  OEM 전략:   #8B5CF6 (Purple)                   │
│  기술 트렌드: #06B6D4 (Cyan)                    │
│  정책/규제:  #F59E0B (Amber)                    │
│  경쟁 동향:  #EF4444 (Red)                      │
│  고객 인사이트: #22C55E (Green)                  │
└─────────────────────────────────────────────────┘
```

### 타이포그래피

```
┌──────────┬──────────────────────────────────────────┐
│ Role     │ Spec                                     │
├──────────┼──────────────────────────────────────────┤
│ H1       │ 28px / 700 / 1.3 / Primary-900           │
│ H2       │ 22px / 600 / 1.4 / Primary-800           │
│ H3       │ 18px / 600 / 1.4 / Primary-700           │
│ Body     │ 15px / 400 / 1.6 / Primary-600           │
│ Body-sm  │ 14px / 400 / 1.5 / Primary-600           │
│ Caption  │ 13px / 400 / 1.4 / Primary-400           │
│ Badge    │ 12px / 700 / 1.0 / White                 │
│ Stat     │ 32px / 700 / 1.2 / Primary-900 (수치)    │
│ Stat-label│ 13px / 500 / 1.3 / Primary-400 (라벨)   │
└──────────┴──────────────────────────────────────────┘

Font Stack: 'Pretendard', 'Noto Sans KR', system-ui, sans-serif
Monospace:  'JetBrains Mono', 'Fira Code', monospace
```

### 간격 시스템 (Spacing Scale)

```
--space-1:   4px    (인접 요소 최소 간격)
--space-2:   8px    (배지 내부, 아이콘 간격)
--space-3:  12px    (리스트 항목 간)
--space-4:  16px    (카드 내부 패딩)
--space-5:  20px    (소섹션 간)
--space-6:  24px    (카드 패딩, 컴포넌트 간)
--space-8:  32px    (섹션 내 그룹 간)
--space-10: 40px    (주요 섹션 간)
--space-12: 48px    (페이지 마진)
--space-16: 64px    (섹션 간 대간격)
```

### 그림자 (Elevation)

```
--shadow-xs:  0 1px 2px rgba(0,0,0,0.05)         (배지, 태그)
--shadow-sm:  0 1px 3px rgba(0,0,0,0.1)           (카드 기본)
--shadow-md:  0 4px 6px rgba(0,0,0,0.07)          (카드 호버)
--shadow-lg:  0 10px 15px rgba(0,0,0,0.1)         (모달, 팝업)
--shadow-xl:  0 20px 25px rgba(0,0,0,0.1)         (플로팅 요소)
```

### 라운드 (Border Radius)

```
--radius-sm:   4px   (배지, 태그)
--radius-md:   8px   (카드, 버튼)
--radius-lg:  12px   (큰 카드, 섹션)
--radius-xl:  16px   (히어로 섹션)
--radius-full: 50%   (원형 배지)
```

---

## 컴포넌트 스펙

### 1. Score Badge (신뢰도 점수 표시)

```
┌────────┐
│  85    │  ← 원형 배지, 점수 숫자
│ /100   │  ← 분모 표시 (선택)
└────────┘

Size: 48x48px (large), 36x36px (medium), 24x24px (small)
Border-radius: 50%
Font: Badge weight

Color Rules:
- 80~100: --color-success  (초록)
- 60~79:  --color-warning  (주황)
- 40~59:  --color-danger   (빨강)
- 0~39:   --color-primary-400 (회색)
```

### 2. Item Card (항목 카드)

```
┌─────────────────────────────────────────────┐
│  ┌──────┐  ┌────────────┐  ┌──────────┐   │
│  │ #1   │  │ 기술 트렌드 │  │ 85/100 │   │ ← 순위 + 카테고리 + 점수
│  └──────┘  └────────────┘  └──────────┘   │
│                                             │
│  SDV 전환 가속에 따른 소프트웨어 부품 수요 급증  │ ← 제목 (H3)
│                                             │
│  핵심 요약 텍스트가 여기에 표시됩니다.       │ ← 2~3줄 요약
│  최대 3줄까지 표시하며 초과 시 truncate.      │
│                                             │
│  ┌───────────────────────────────────────┐  │
│  │ 💡 시사점: OEM에게 SDV 관련...        │  │ ← 시사점 박스
│  └───────────────────────────────────────┘  │
│                                             │
│  📰 Automotive News · 2025-01-15           │ ← 출처 + 날짜
└─────────────────────────────────────────────┘

Specs:
- Width: 100% (single col) or calc(50% - gap) (2-col)
- Padding: var(--space-6)
- Border-left: 4px solid [category-color]
- Background: var(--color-card)
- Shadow: var(--shadow-sm)
- Hover: var(--shadow-md) + translateY(-2px)
- Border-radius: var(--radius-md)
```

### 3. Stats Card (핵심 수치 카드)

```
┌─────────────────┐
│      85         │ ← Stat 크기 수치
│  평균 신뢰도     │ ← Stat-label
│     /100        │ ← 부가 설명
└─────────────────┘

Specs:
- Min-width: 140px
- Padding: var(--space-6)
- Text-align: center
- Background: var(--color-card)
- Border-radius: var(--radius-md)
- Shadow: var(--shadow-sm)
```

### 4. Category Tag (영역 태그)

```
┌──────────────┐
│ 🔧 기술 트렌드 │
└──────────────┘

Specs:
- Padding: 2px 8px
- Border-radius: var(--radius-sm)
- Font: Badge
- Background: [category-color] with 15% opacity
- Color: [category-color] darkened
- Border: 1px solid [category-color] with 30% opacity
```

### 5. Hero Card (최상위 인사이트)

```
┌─────────────────────────────────────────────────┐
│  🔥 #1 핵심 인사이트                             │
│                                                  │
│  [큰 글씨로 핵심 한 문장]                         │
│                                                  │
│  ┌─────┐                                        │
│  │ 92  │  출처: McKinsey | 2025-01-20           │
│  └─────┘                                        │
└─────────────────────────────────────────────────┘

Specs:
- Background: linear-gradient(135deg, --color-primary-800, --color-primary-700)
- Color: white
- Padding: var(--space-8)
- Border-radius: var(--radius-xl)
- Shadow: var(--shadow-lg)
```

### 6. Insight Box (시사점/액션 박스)

```
┌─ 💡 ────────────────────────────────────────────┐
│  What: OEM이 SDV 전환을 1년 앞당겨 추진 중      │
│  So What: SW 부품 공급사에게 조기 RFQ 기회 발생  │
│  Now What: 현대차 전자기획팀 2월 RFQ 대비 필요   │
└─────────────────────────────────────────────────┘

Specs:
- Background: --color-info with 5% opacity
- Border-left: 3px solid --color-info
- Padding: var(--space-4)
- Border-radius: var(--radius-sm)
- Font: Body-sm
```

---

## 레이아웃 패턴

### 전체 페이지 구조

```
Max-width: 1200px
Margin: 0 auto
Padding-x: var(--space-6) (mobile) / var(--space-12) (desktop)

Sections (top to bottom):
1. Header        - 전체 너비, 그라데이션 배경
2. Stats Bar     - 3~4 컬럼 그리드
3. Hero Section  - 1~3 컬럼 (탑 인사이트)
4. Item Grid     - 1~2 컬럼 (10개 항목 카드)
5. Coverage      - 테이블 or 매트릭스 시각화
6. Insights      - 2컬럼 (기회 | 위협)
7. Footer        - 전체 너비, 다크 배경
```

### 반응형 그리드

```
Desktop (≥1024px):
- Stats: 4-column grid
- Heroes: 3-column grid
- Items: 2-column grid
- Insights: 2-column grid

Tablet (768~1023px):
- Stats: 2-column grid
- Heroes: 1-column stack (carousel 가능)
- Items: 1-column full-width
- Insights: 1-column stack

Mobile (<768px):
- All: 1-column stack
- Stats: 2-column grid (축소)
- Cards: 축소된 패딩, 작은 타이포
```

---

## 인쇄 최적화

```css
@media print {
    /* 배경색 제거 */
    * { background: white !important; }
    
    /* 그림자 제거, 보더로 대체 */
    .card { box-shadow: none; border: 1px solid #ddd; }
    
    /* 페이지 나눔 방지 */
    .item-card { break-inside: avoid; }
    
    /* 색상 → 흑백 대비 유지 */
    .score-badge { border: 2px solid currentColor; }
    
    /* 불필요 요소 숨김 */
    .hover-effects, .animations { display: none; }
    
    /* 폰트 크기 축소 */
    body { font-size: 12px; }
    h1 { font-size: 22px; }
}
```

---

## 접근성 규칙

1. **색상 대비**: 텍스트 vs 배경 최소 4.5:1 (WCAG AA)
2. **폰트 크기**: 본문 최소 14px, 캡션 최소 12px
3. **터치 영역**: 인터랙티브 요소 최소 44x44px
4. **의미 전달**: 색상만으로 정보 전달하지 않음 (아이콘/텍스트 병행)
5. **시맨틱**: 적절한 heading 계층, aria-label 사용

---

## 성능 가이드라인

- 단일 HTML 파일 최대 크기: 200KB
- 인라인 SVG 사용 (외부 이미지 금지)
- CSS 변수로 반복 제거
- 불필요한 애니메이션 금지 (hover transition만 허용, 0.2s ease)
- Google Fonts 사용 시 `display=swap` 필수
