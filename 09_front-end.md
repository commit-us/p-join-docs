# P Join · 프런트엔드 스타일

데스크톱 Figma 시안에 맞춰 공통 페이지의 여백, 폰트, 색상을 적용합니다. 화면별 예외는 [IA 디자인 시스템](02_ia.md#design-system)과 [페이지별 화면 명세](02_ia.md#screen-specs)를 참고합니다. 기준 시안은 [S01 프로젝트 목록](https://www.figma.com/design/5vNU5XRiZapt6pg2AbvwrZ/P-Join?node-id=2-3)과 [S09 프로젝트 설정](https://www.figma.com/design/5vNU5XRiZapt6pg2AbvwrZ/P-Join?node-id=2-218)입니다.

## Body와 콘텐츠 여백

| 항목 | 데스크톱 기준 |
| --- | --- |
| 브라우저 기본 `body` 여백 | CSS에서 `margin: 0`으로 초기화 |
| Figma 프레임 너비 | `1440px` |
| 헤더·본문 콘텐츠 최대 폭 | `1240px`, 가로 중앙 정렬 |
| 좌우 여백 | `1440px`에서 각각 `100px` |
| 헤더 | 높이 `76px`, 하단 구분선 `1px` |
| 본문 시작 | 상단 기준선에서 `48px` 아래. 화면별 시안의 예외를 우선 적용 |
| 카드 안쪽 여백·카드 사이 간격 | 각각 `24px` |

헤더 로고와 본문의 왼쪽 시작선을 맞추고, 헤더 프로필과 본문의 오른쪽 끝선을 맞춥니다. 프레임 높이는 콘텐츠에 따라 늘어납니다. 모바일·태블릿의 여백과 브레이크포인트는 시안 확정 후 반영합니다.

## 폰트

페이지와 모달의 한글·영문·숫자는 **Pretendard**를 사용합니다. 버튼과 입력 요소에도 본문의 폰트를 상속합니다.

| 굵기 | CSS 값 |
| --- | --- |
| Regular | `400` |
| Medium | `500` |
| SemiBold | `600` |
| Bold | `700` |

| 텍스트 역할 | 시안 기준 크기 |
| --- | --- |
| 페이지 제목 | `32px` |
| 카드 제목 | `16px` |
| 본문·필드 라벨 | `13px` |
| 도움말 | `11–12px` |

글자 굵기와 줄 높이는 화면별 Figma 텍스트 스타일을 따릅니다.

## 색상

| 역할 | 값 | 적용 위치 |
| --- | --- | --- |
| 기본 배경 | `#FFFFFF` | 페이지·카드·보조 버튼 |
| 기본 텍스트 | `#1C1C1C` | 제목·본문·활성 라벨 |
| 보조 텍스트 | `#777777` | 설명·메타 정보·비활성 보조 문구 |
| 구분선·윤곽선 | `#E5E5E5` | 헤더·카드·필드·보조 버튼 |
| 비활성 배경 | `#F0F0F0` | 비활성 버튼 등 상태 표시 |
| 주요 버튼 | 배경 `#1C1C1C`, 글자 `#FFFFFF` | 활성 주요 버튼 |
| Accent/100 | `#002CFA` | 강조 텍스트·버튼 윤곽선·활성 북마크 아이콘 |
| Accent/10 | `#002CFA1A` | 불투명도 10%의 파생 토큰. 시안에는 토큰 정의만 유지 |
| 모달 딤 | `rgba(28, 31, 41, 0.38)` | 열린 모달 뒤 전체 화면 |

활성 좋아요 하트는 빨강, 미확인 알림 배지는 빨강 바탕에 흰색 숫자를 사용합니다. 기술 스택 로고는 공식 브랜드 색을 유지합니다.

## 공통 CSS 적용 예시

`page-content`는 헤더 내부와 본문에 공통으로 적용합니다. Pretendard는 프로젝트에서 먼저 로드합니다.

```css
:root {
  --page-max-width: 1240px;
  --color-background: #ffffff;
  --color-text: #1c1c1c;
  --color-text-muted: #777777;
  --color-border: #e5e5e5;
  --color-disabled-background: #f0f0f0;
  --color-accent: #002cfa;
  --color-accent-10: #002cfa1a;
  --color-modal-overlay: rgba(28, 31, 41, 0.38);
}

body {
  margin: 0;
  background-color: var(--color-background);
  color: var(--color-text);
  font-family: Pretendard, sans-serif;
}

button,
input,
select,
textarea {
  font: inherit;
}

.page-content {
  width: 100%;
  max-width: var(--page-max-width);
  margin-inline: auto;
}
```
