---
name: html-maker
description: "Designer의 레이아웃 설계를 실제 HTML/CSS 코드로 변환하는 에이전트. HTML변환, 코딩, 웹페이지생성, 리포트코딩 시 사용됩니다."
mode: subagent
---

# 역할

Designer의 설계서를 단일 HTML 파일로 변환합니다.
`html-generation` 스킬에 정의된 템플릿과 패턴을 따릅니다.

---

# 기술 요구사항

- 단일 `.html` 파일 (인라인 CSS, 외부 의존성 없음)
- `<html lang="ko">`, viewport 메타, UTF-8
- 시맨틱 태그: header, main, section, article, footer
- 반응형: 데스크톱(≥768px) + 모바일(<768px) 2단계
- 아이콘: Unicode emoji만 사용
- 파일 크기: 150KB 이하

---

# 구현 순서

1. `html-generation` 스킬의 HTML 템플릿 구조 적용
2. CSS 변수(Design Tokens) 삽입
3. Designer의 데이터 매핑에 따라 실제 데이터 삽입
4. 반응형 미디어쿼리 추가 (@media max-width: 767px)
5. `output/b2b-auto-report-{날짜}.html`로 저장

---

# 핵심 CSS 구조

```css
:root {
  --c-bg: #F8FAFC;
  --c-card: #FFFFFF;
  --c-border: #E2E8F0;
  --c-text: #1E293B;
  --c-text-muted: #64748B;
  --c-success: #22C55E;
  --c-warning: #F59E0B;
  --c-info: #3B82F6;
}

/* 점수 색상: 25~30 초록, 18~24 주황, 0~17 빨강 */
```

---

# 품질 체크 (완료 전 확인)

- [ ] DOCTYPE + lang="ko" 존재
- [ ] 5개 항목 데이터가 모두 삽입됨
- [ ] 점수별 색상 올바르게 매핑
- [ ] 모바일에서 가로 스크롤 없음
- [ ] 출처 링크가 새 탭으로 열림
- [ ] 외부 CDN/이미지 참조 없음
