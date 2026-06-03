---
name: report-design
description: "B2B 시장 분석 리포트의 시각 디자인 원칙과 레이아웃 가이드. 정보 계층, 색상 시스템, 타이포그래피, 컴포넌트 스펙을 정의합니다. 리포트디자인, 레이아웃, 시각화, UI설계 시 사용됩니다."
---

# B2B 시장 분석 리포트 디자인 가이드

## 개요

자동차 B2B 영업팀이 빠르게 소화할 수 있는 시각적 리포트를 설계하기 위한 디자인 원칙과 컴포넌트 라이브러리입니다.

---

## 디자인 철학

### 1. Information-First Design

- 데코레이션이 아닌 **정보 전달**이 디자인의 목적
- 모든 시각 요소는 정보 이해를 돕는 역할이 있어야 함
- 장식적 요소 사용 금지 (그라데이션, 그림자는 기능적 용도에만)

### 2. Progressive Disclosure

- **Level 1** (3초): 핵심 인사이트 3개 (Hero Section)
- **Level 2** (30초): 전체 10개 항목 요약 스캔 (카드 그리드)
- **Level 3** (3분): 개별 항목 상세 분석 (카드 내부)
- 독자가 원하는 깊이까지만 읽을 수 있는 구조

### 3. Consistency Over Creativity

- 매주 발행되는 정기 리포트이므로 **일관된 포맷** 유지
- 독자가 "어디에 무엇이 있는지" 학습할 수 있어야 함
- 이번 주와 지난 주 리포트의 구조가 동일

---

## 레이아웃 시스템

### 그리드

```
전체 너비: max-width 1200px (중앙 정렬)
여백: 좌우 24px (모바일), 48px (데스크톱)
그리드: 12컬럼 (거터 24px)
```

### 섹션 순서 (고정)

```
1. Header (제목 + 메타)
2. Hero Section (Top 3 인사이트)
3. Stats Bar (KPI 수치)
4. Item Grid (10개 상세 카드)
5. Coverage Map (영역 분포)
6. Footer (면책 + 메타데이터)
```

### 섹션 간 간격

- 섹션 간: 48px
- 섹션 내부 요소 간: 24px
- 카드 내부: 16px

---

## 컴포넌트 라이브러리

### 1. Hero Card

**용도**: Top 3 핵심 인사이트 강조

```
크기: 1/3 너비 (데스크톱), 풀와이드 (모바일)
패딩: 24px
모서리: 12px radius
배경: 순위별 강조색 그라데이션
    1위: linear-gradient(135deg, #1E40AF, #3B82F6)
    2위: linear-gradient(135deg, #1E3A5F, #60A5FA)
    3위: linear-gradient(135deg, #1E293B, #94A3B8)
텍스트: 흰색
```

**내부 구조**:
- 영역 태그 (뱃지, 12px)
- 핵심 문장 (18px, Bold)
- 신뢰도 바 (시각적 게이지)
- 출처 + 날짜 (Caption)

### 2. Stats Card

**용도**: 핵심 수치 KPI 표시

```
크기: 1/5 너비 (데스크톱), 2/5 (태블릿), 풀와이드 (모바일)
패딩: 16px
모서리: 8px radius
배경: 흰색
테두리: 1px solid var(--color-border)
```

**내부 구조**:
- 라벨 (Caption, Secondary 색상)
- 수치 (H2, Bold, Primary 색상)
- 변화 표시 (Optional, 증감 아이콘)

### 3. Item Card

**용도**: 개별 리서치 항목 표시

```
크기: 1/2 너비 (데스크톱), 풀와이드 (모바일)
패딩: 24px
모서리: 10px radius
배경: 흰색
테두리: 1px solid var(--color-border)
호버: box-shadow 0 4px 12px rgba(0,0,0,0.08)
```

**내부 구조**:
```
┌──────────────────────────────────────┐
│ [#순위 뱃지]  [영역 태그]            │
│                                      │
│ 제목 (H3, Bold)                      │
│ ──────────────────────────────────── │
│ 핵심 내용 (Body, 3줄)                │
│                                      │
│ ┌─ 시사점 박스 (배경: Slate-50) ──┐ │
│ │ What: ...                        │ │
│ │ So What: ...                     │ │
│ │ Now What: ... (Bold 강조)        │ │
│ └──────────────────────────────────┘ │
│                                      │
│ 신뢰도: ████████░░ 82/100           │
│ 출처: {링크}  |  {날짜}              │
└──────────────────────────────────────┘
```

### 4. Reliability Bar

**용도**: 신뢰도 점수 시각화

```
높이: 8px
배경: Slate-200
채움: 점수 비례 너비
색상: 점수별 자동 매핑
    80-100: Green-500
    60-79: Amber-500
    0-59: Red-500
모서리: 4px radius
```

### 5. Area Tag (영역 뱃지)

**용도**: 수집 영역 분류 표시

```
패딩: 4px 10px
모서리: 12px radius (pill shape)
폰트: 12px, Medium
```

**영역별 색상**:
| 영역 | 배경 | 텍스트 |
|------|------|--------|
| 시장 동향 | Blue-100 | Blue-700 |
| OEM 전략 | Purple-100 | Purple-700 |
| 기술 트렌드 | Emerald-100 | Emerald-700 |
| 정책/규제 | Amber-100 | Amber-700 |
| 경쟁 동향 | Rose-100 | Rose-700 |
| 고객 인사이트 | Cyan-100 | Cyan-700 |

### 6. Coverage Map

**용도**: 6개 영역 커버리지 시각화

```
형태: 수평 바 차트
높이: 각 바 32px
간격: 바 간 12px
최대 너비: 컨테이너 100%
바 채움: 해당 영역 항목 수 / 전체 × 100%
라벨: 좌측 영역명, 우측 항목 수
```

---

## 색상 시스템 (Design Tokens)

```css
/* Primary Palette */
--color-primary-50: #EFF6FF;
--color-primary-100: #DBEAFE;
--color-primary-500: #3B82F6;
--color-primary-600: #2563EB;
--color-primary-700: #1D4ED8;

/* Semantic Colors */
--color-success: #22C55E;
--color-warning: #F59E0B;
--color-danger: #EF4444;

/* Neutral Palette */
--color-slate-50: #F8FAFC;
--color-slate-100: #F1F5F9;
--color-slate-200: #E2E8F0;
--color-slate-500: #64748B;
--color-slate-800: #1E293B;
--color-slate-900: #0F172A;
```

---

## 타이포그래피 스케일

```css
--text-xs: 12px / 16px;    /* Badge, Micro label */
--text-sm: 13px / 20px;    /* Caption, Meta */
--text-base: 15px / 24px;  /* Body */
--text-lg: 18px / 28px;    /* Card title, H3 */
--text-xl: 22px / 30px;    /* Section title, H2 */
--text-2xl: 28px / 36px;   /* Page title, H1 */
--text-3xl: 36px / 44px;   /* Hero number */
```

---

## 반응형 전략

| 브레이크포인트 | 적용 |
|---------------|------|
| < 640px (Mobile) | 1컬럼, 패딩 축소, Hero 세로 배치 |
| 640~1023px (Tablet) | Hero 3컬럼 유지, Item 1컬럼 |
| ≥ 1024px (Desktop) | Hero 3컬럼, Item 2컬럼, Stats 5컬럼 |

---

## 접근성 기준

- 색상 대비: WCAG AA (4.5:1 텍스트, 3:1 대형 텍스트)
- 색상만으로 정보 전달하지 않음 (텍스트 라벨 병행)
- 포커스 표시 명확 (링크, 인터랙티브 요소)
- 의미 있는 마크업 순서 (스크린 리더 호환)

---

## 제약 사항

- 외부 이미지/아이콘 CDN 사용 금지
- 아이콘은 Unicode emoji 또는 CSS로 구현
- 그래프/차트는 CSS로 구현 (Canvas/SVG 라이브러리 금지)
- 인쇄 시 색상 의존 정보는 텍스트로도 전달되어야 함
