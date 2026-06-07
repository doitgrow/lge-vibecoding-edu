---
name: report-design-system
description: "비즈니스 리포트의 시각 디자인 시스템. 색상 팔레트, 타이포그래피, 컴포넌트 규격, 레이아웃 패턴을 정의합니다. 리포트디자인, 시각화, UI설계, HTML리포트 시 사용됩니다."
---

# 리포트 디자인 시스템 (간소화 버전)

> 이 스킬은 `report-design` 스킬과 함께 사용됩니다.
> 상세 색상/타이포/레이아웃은 `report-design`을 참조하세요.

---

## CSS 변수 (복사하여 사용)

```css
:root {
  --c-bg: #F8FAFC;
  --c-card: #FFFFFF;
  --c-border: #E2E8F0;
  --c-text: #1E293B;
  --c-text-body: #475569;
  --c-text-muted: #64748B;
  --c-text-caption: #94A3B8;
  --c-success: #22C55E;
  --c-warning: #F59E0B;
  --c-danger: #EF4444;
  --c-info: #3B82F6;
  --c-cat-market: #3B82F6;
  --c-cat-oem: #8B5CF6;
  --c-cat-tech: #06B6D4;
  --c-cat-policy: #F59E0B;
}
```

---

## 점수별 색상 규칙

| 점수 범위 | 색상 | 의미 |
|-----------|------|------|
| 25~30 | var(--c-success) 초록 | 높은 품질 |
| 18~24 | var(--c-warning) 주황 | 양호 |
| 0~17 | var(--c-danger) 빨강 | 미흡 |

---

## 카테고리별 왼쪽 보더 색상

| 카테고리 | 색상 |
|----------|------|
| 시장 동향 | var(--c-cat-market) #3B82F6 |
| OEM 전략 | var(--c-cat-oem) #8B5CF6 |
| 기술 트렌드 | var(--c-cat-tech) #06B6D4 |
| 경쟁/정책 | var(--c-cat-policy) #F59E0B |

---

## 간격 규칙

- 섹션 간: 32px
- 카드 간: 16px
- 카드 내부 패딩: 20px
- 요소 간: 12px
