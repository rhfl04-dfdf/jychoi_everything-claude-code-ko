# 스타일 프리셋 참고자료

`frontend-slides`를 위한 선별된 시각적 스타일 모음입니다.

이 파일의 용도:
- 필수 뷰포트 맞춤 CSS 베이스
- 프리셋 선택 및 분위기 매핑
- CSS 주의사항 및 검증 규칙

추상적인 도형만 사용합니다. 사용자가 명시적으로 요청하지 않는 한 일러스트레이션을 피하세요.

## 뷰포트 맞춤은 절대 원칙

모든 슬라이드는 하나의 뷰포트에 완전히 맞아야 합니다.

### 황금 법칙

```text
Each slide = exactly one viewport height.
Too much content = split into more slides.
Never scroll inside a slide.
```

### 밀도 제한

| 슬라이드 유형 | 최대 콘텐츠 |
|------------|-----------------|
| 제목 슬라이드 | 제목 1개 + 부제목 1개 + 선택적 태그라인 |
| 콘텐츠 슬라이드 | 제목 1개 + 불릿 4-6개 또는 단락 2개 |
| 기능 그리드 | 카드 최대 6개 |
| 코드 슬라이드 | 최대 8-10줄 |
| 인용 슬라이드 | 인용 1개 + 출처 |
| 이미지 슬라이드 | 이미지 1개, 이상적으로 60vh 이하 |

## 필수 베이스 CSS

모든 생성된 프레젠테이션에 이 블록을 복사하고 그 위에 테마를 적용하세요.

```css
/* ===========================================
   VIEWPORT FITTING: MANDATORY BASE STYLES
   =========================================== */

html, body {
    height: 100%;
    overflow-x: hidden;
}

html {
    scroll-snap-type: y mandatory;
    scroll-behavior: smooth;
}

.slide {
    width: 100vw;
    height: 100vh;
    height: 100dvh;
    overflow: hidden;
    scroll-snap-align: start;
    display: flex;
    flex-direction: column;
    position: relative;
}

.slide-content {
    flex: 1;
    display: flex;
    flex-direction: column;
    justify-content: center;
    max-height: 100%;
    overflow: hidden;
    padding: var(--slide-padding);
}

:root {
    --title-size: clamp(1.5rem, 5vw, 4rem);
    --h2-size: clamp(1.25rem, 3.5vw, 2.5rem);
    --h3-size: clamp(1rem, 2.5vw, 1.75rem);
    --body-size: clamp(0.75rem, 1.5vw, 1.125rem);
    --small-size: clamp(0.65rem, 1vw, 0.875rem);

    --slide-padding: clamp(1rem, 4vw, 4rem);
    --content-gap: clamp(0.5rem, 2vw, 2rem);
    --element-gap: clamp(0.25rem, 1vw, 1rem);
}

.card, .container, .content-box {
    max-width: min(90vw, 1000px);
    max-height: min(80vh, 700px);
}

.feature-list, .bullet-list {
    gap: clamp(0.4rem, 1vh, 1rem);
}

.feature-list li, .bullet-list li {
    font-size: var(--body-size);
    line-height: 1.4;
}

.grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(min(100%, 250px), 1fr));
    gap: clamp(0.5rem, 1.5vw, 1rem);
}

img, .image-container {
    max-width: 100%;
    max-height: min(50vh, 400px);
    object-fit: contain;
}

@media (max-height: 700px) {
    :root {
        --slide-padding: clamp(0.75rem, 3vw, 2rem);
        --content-gap: clamp(0.4rem, 1.5vw, 1rem);
        --title-size: clamp(1.25rem, 4.5vw, 2.5rem);
        --h2-size: clamp(1rem, 3vw, 1.75rem);
    }
}

@media (max-height: 600px) {
    :root {
        --slide-padding: clamp(0.5rem, 2.5vw, 1.5rem);
        --content-gap: clamp(0.3rem, 1vw, 0.75rem);
        --title-size: clamp(1.1rem, 4vw, 2rem);
        --body-size: clamp(0.7rem, 1.2vw, 0.95rem);
    }

    .nav-dots, .keyboard-hint, .decorative {
        display: none;
    }
}

@media (max-height: 500px) {
    :root {
        --slide-padding: clamp(0.4rem, 2vw, 1rem);
        --title-size: clamp(1rem, 3.5vw, 1.5rem);
        --h2-size: clamp(0.9rem, 2.5vw, 1.25rem);
        --body-size: clamp(0.65rem, 1vw, 0.85rem);
    }
}

@media (max-width: 600px) {
    :root {
        --title-size: clamp(1.25rem, 7vw, 2.5rem);
    }

    .grid {
        grid-template-columns: 1fr;
    }
}

@media (prefers-reduced-motion: reduce) {
    *, *::before, *::after {
        animation-duration: 0.01ms !important;
        transition-duration: 0.2s !important;
    }

    html {
        scroll-behavior: auto;
    }
}
```

## 뷰포트 체크리스트

- 모든 `.slide`에 `height: 100vh`, `height: 100dvh`, `overflow: hidden`이 있음
- 모든 타이포그래피가 `clamp()`를 사용함
- 모든 간격이 `clamp()` 또는 뷰포트 단위를 사용함
- 이미지에 `max-height` 제약이 있음
- 그리드가 `auto-fit` + `minmax()`로 적응함
- 단축 높이 브레이크포인트가 `700px`, `600px`, `500px`에 존재함
- 콘텐츠가 답답하게 느껴지면 슬라이드를 분할함

## 분위기와 프리셋 매핑

| 분위기 | 적합한 프리셋 |
|------|--------------|
| 인상적 / 자신감 있는 | Bold Signal, Electric Studio, Dark Botanical |
| 흥분 / 활기찬 | Creative Voltage, Neon Cyber, Split Pastel |
| 차분 / 집중된 | Notebook Tabs, Paper & Ink, Swiss Modern |
| 영감을 주는 / 감동적인 | Dark Botanical, Vintage Editorial, Pastel Geometry |

## 프리셋 카탈로그

### 1. Bold Signal

- 분위기: 자신감 있고, 강렬하며, 키노트 준비 완료
- 최적 용도: 피치 덱, 런치, 선언문
- 폰트: Archivo Black + Space Grotesk
- 팔레트: 차콜 베이스, 핫 오렌지 포컬 카드, 크리스프 화이트 텍스트
- 시그니처: 큰 섹션 번호, 어두운 배경 위 고대비 카드

### 2. Electric Studio

- 분위기: 깔끔하고, 굵직하며, 에이전시 수준의 완성도
- 최적 용도: 클라이언트 프레젠테이션, 전략적 리뷰
- 폰트: Manrope 전용
- 팔레트: 검정, 흰색, 채도 높은 코발트 액센트
- 시그니처: 투 패널 분할과 날카로운 에디토리얼 정렬

### 3. Creative Voltage

- 분위기: 활기차고, 레트로-모던하며, 유쾌한 자신감
- 최적 용도: 크리에이티브 스튜디오, 브랜드 작업, 제품 스토리텔링
- 폰트: Syne + Space Mono
- 팔레트: 일렉트릭 블루, 네온 옐로우, 딥 네이비
- 시그니처: 하프톤 텍스처, 배지, 강렬한 대비

### 4. Dark Botanical

- 분위기: 우아하고, 프리미엄이며, 분위기 있는
- 최적 용도: 럭셔리 브랜드, 사려 깊은 내러티브, 프리미엄 제품 덱
- 폰트: Cormorant + IBM Plex Sans
- 팔레트: 거의 검정색, 따뜻한 아이보리, 블러시, 골드, 테라코타
- 시그니처: 흐릿한 추상적 원, 얇은 선, 절제된 모션

### 5. Notebook Tabs

- 분위기: 에디토리얼하고, 정돈되며, 촉각적인
- 최적 용도: 보고서, 리뷰, 구조화된 스토리텔링
- 폰트: Bodoni Moda + DM Sans
- 팔레트: 차콜 위 크림 종이에 파스텔 탭
- 시그니처: 종이 시트, 색상 있는 사이드 탭, 바인더 디테일

### 6. Pastel Geometry

- 분위기: 친근하고, 모던하며, 우호적인
- 최적 용도: 제품 개요, 온보딩, 가벼운 브랜드 덱
- 폰트: Plus Jakarta Sans 전용
- 팔레트: 연한 파란 배경, 크림 카드, 소프트 핑크/민트/라벤더 액센트
- 시그니처: 수직 필, 둥근 카드, 부드러운 그림자

### 7. Split Pastel

- 분위기: 유쾌하고, 모던하며, 창의적인
- 최적 용도: 에이전시 소개, 워크숍, 포트폴리오
- 폰트: Outfit 전용
- 팔레트: 피치 + 라벤더 분할에 민트 배지
- 시그니처: 분할 배경, 둥근 태그, 밝은 그리드 오버레이

### 8. Vintage Editorial

- 분위기: 위트 있고, 개성 있으며, 매거진에서 영감을 받은
- 최적 용도: 개인 브랜드, 의견이 담긴 발표, 스토리텔링
- 폰트: Fraunces + Work Sans
- 팔레트: 크림, 차콜, 먼지 낀 따뜻한 액센트
- 시그니처: 기하학적 액센트, 테두리 있는 콜아웃, 강렬한 세리프 헤드라인

### 9. Neon Cyber

- 분위기: 미래적이고, 테크적이며, 역동적인
- 최적 용도: AI, 인프라, 개발 도구, 미래형 발표
- 폰트: Clash Display + Satoshi
- 팔레트: 미드나이트 네이비, 시안, 마젠타
- 시그니처: 글로우, 파티클, 그리드, 데이터 레이더 에너지

### 10. Terminal Green

- 분위기: 개발자 중심, 해커 클린
- 최적 용도: API, CLI 도구, 엔지니어링 데모
- 폰트: JetBrains Mono 전용
- 팔레트: GitHub 다크 + 터미널 그린
- 시그니처: 스캔 라인, 커맨드라인 프레이밍, 정밀한 모노스페이스 리듬

### 11. Swiss Modern

- 분위기: 미니멀하고, 정밀하며, 데이터 중심
- 최적 용도: 기업, 제품 전략, 분석
- 폰트: Archivo + Nunito
- 팔레트: 흰색, 검정, 시그널 레드
- 시그니처: 보이는 그리드, 비대칭, 기하학적 규율

### 12. Paper & Ink

- 분위기: 문학적이고, 사려 깊으며, 이야기 중심
- 최적 용도: 에세이, 키노트 내러티브, 선언문 덱
- 폰트: Cormorant Garamond + Source Serif 4
- 팔레트: 따뜻한 크림, 차콜, 크림슨 액센트
- 시그니처: 풀 인용, 드롭 캡, 우아한 선

## 직접 선택 프롬프트

사용자가 이미 원하는 스타일을 알고 있다면, 미리보기 생성을 강제하는 대신 위의 프리셋 이름에서 직접 선택하게 하세요.

## 애니메이션 느낌 매핑

| 느낌 | 모션 방향 |
|---------|------------------|
| 드라마틱 / 시네마틱 | 느린 페이드, 패럴랙스, 큰 스케일인 |
| 테크 / 미래적 | 글로우, 파티클, 그리드 모션, 텍스트 스크램블 |
| 유쾌 / 친근한 | 스프링 이징, 둥근 도형, 플로팅 모션 |
| 전문적 / 기업적 | 200-300ms의 섬세한 전환, 깔끔한 슬라이드 |
| 차분 / 미니멀 | 매우 절제된 움직임, 여백 우선 |
| 에디토리얼 / 매거진 | 강한 계층구조, 텍스트와 이미지의 엇갈린 상호작용 |

## CSS 주의사항: 함수 부정

다음과 같이 작성하지 마세요:

```css
right: -clamp(28px, 3.5vw, 44px);
margin-left: -min(10vw, 100px);
```

브라우저가 조용히 무시합니다.

대신 항상 이렇게 작성하세요:

```css
right: calc(-1 * clamp(28px, 3.5vw, 44px));
margin-left: calc(-1 * min(10vw, 100px));
```

## 검증 크기

최소한 다음 크기에서 테스트하세요:
- 데스크톱: `1920x1080`, `1440x900`, `1280x720`
- 태블릿: `1024x768`, `768x1024`
- 모바일: `375x667`, `414x896`
- 가로 방향 폰: `667x375`, `896x414`

## 안티패턴

다음을 사용하지 마세요:
- 흰 배경에 보라색 스타트업 템플릿
- 사용자가 명시적으로 유틸리티적 중립성을 원하지 않는 한 Inter / Roboto / Arial을 시각적 목소리로 사용
- 불릿 벽, 작은 타입, 스크롤이 필요한 코드 블록
- 추상적 기하학이 더 잘할 수 있는 경우의 장식적 일러스트레이션
