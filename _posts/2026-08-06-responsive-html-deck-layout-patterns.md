---
title: "PPT가 아니라 웹페이지처럼 보이는 반응형 발표자료 만들기"
date: 2026-08-06 17:00:00 +0900
categories: [Frontend]
tags: [html, css, presentation, layout, responsive]
---

발표자료 자동 생성 도구를 만들다가 한동안 붙잡고 있던 질문이 있다. PPTX로 뽑으면 그 자리에서 끝나는데, 굳이 HTML 출력을 따로 설계할 이유가 있나.

처음엔 없다고 봤다. 그런데 HTML 쪽을 붙잡고 며칠 지나니까 생각이 조금 뒤집혔다. PPT는 콘텐츠를 슬라이드 안에 쌓는다. 텍스트 상자, 이미지, 도형이 한 판에 얹히는 구조다. HTML은 그렇지 않다. 슬라이드마다 **레이아웃 패턴 자체를 갈아끼울 수 있다**. 표지 슬라이드가 hero 패턴이면, 다음 슬라이드는 격자, 그 다음은 좌우 2단 비교. 이걸 슬라이드 단위로 결정할 수 있다는 점이 결정적이었다.

## 왜 색상과 레이아웃을 분리했나

원래는 "테마" 하나로 다 묶으려 했다. `dark-minimal`을 고르면 검은 배경에 단일 컬럼이 딸려오는 식으로. 편해 보였다.

문제는 실제 슬라이드마다 정보 밀도가 다르다는 것이다. 표지는 여백이 왕창 남는 hero가 맞고, 기능 비교는 격자가 낫고, 아키텍처는 본문 옆에 그림이 붙어야 한다. 색상 팔레트는 deck 전체에서 하나면 충분한데, 레이아웃은 슬라이드마다 다르길 원했다. 하나의 축이라고 부르기엔 성질이 너무 달랐다.

그래서 두 축으로 잘랐다.

| 축 | 범위 | 선택 시점 |
|---|---|---|
| Color Style | 배경·글꼴·강조색 | deck 전체 1회 |
| Layout Pattern | 슬라이드 내부 배치 | 슬라이드마다 |

CLI에서는 `--layout`으로 deck 기본값을 잡고, 필요할 때만 슬라이드별로 override한다.

```bash
python make_deck.py input.md --layout bento-grid
python make_deck.py input.md --per-slide-layouts '{"3": "hero", "5": "comparison-matrix"}'
```

Marp(마크다운 기반 발표 도구) 워크플로에서는 슬라이드 주석으로도 바꾼다.

```html
<!-- _class: layout-bento-grid -->
```

## 다섯 개 패턴

일단 다섯 개만 만들었다. `layout-<slug>` 클래스 하나로 켜지고, 내부는 CSS Grid나 Flexbox다.

**hero** — 표지·섹션 구분. 가운데 정렬 하나가 전부다.

```css
.layout-hero {
  display: grid;
  place-items: center;
  text-align: center;
}
```

**bento-grid** — 애플 제품 페이지에서 이름을 가져온 벤토 격자. 카드 크기를 일부러 다르게 줘서 시각적 무게 차이를 만든다. 여러 포인트를 한 슬라이드에 얹을 때 쓴다.

```css
.layout-bento-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-auto-rows: minmax(120px, auto);
  gap: 1rem;
}
.layout-bento-grid .card:first-child {
  grid-column: span 2;
}
```

**magazine-editorial** — 좌측 큰 이미지, 우측 텍스트 블록. 사례 소개용.

**device-mockup** — 브라우저·모바일 프레임 안에 스크린샷을 앉히는 패턴. 상단 바는 `::before`로 그린다. 제품 데모 슬라이드에서만 쓴다.

**comparison-matrix** — 2컬럼(혹은 N컬럼) 비교. before/after, 기술 스택 대치.

```css
.layout-comparison-matrix {
  display: grid;
  grid-template-columns: 1fr 1fr;
  column-gap: 2rem;
}
```

## "반응형"이 발표자료에서 뭘 의미하는가

반응형이라고 하면 모바일이 먼저 떠오른다. 그런데 발표자료의 반응형은 좀 다르다. 실제로 부딪힌 건 두 가지였다.

빔프로젝터 1920×1080과 노트북 화면 1440×900에서 같은 파일을 열었을 때 잘리면 안 된다는 것. 그리고 bento-grid에서 카드가 2개일 때와 5개일 때 둘 다 봐줄 만해야 한다는 것.

앞의 것은 `vw`와 `clamp()`로 잡았다. 뒤는 `auto-fit`으로.

```css
.slide {
  font-size: clamp(0.9rem, 1.8vw, 1.2rem);
  padding: clamp(1.5rem, 4vw, 3rem);
}

.layout-bento-grid {
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
}
```

`clamp()`는 그럭저럭 잘 동작한다. 문제는 `auto-fit`이다. 카드가 1개일 때 그 카드가 화면 전체를 먹어버리는 케이스를 아직 못 잡았다. `max-width`를 걸면 정렬이 이상해지고, `justify-content: start`로 하면 남는 공간이 너무 커진다. 카드 1개짜리 슬라이드는 애초에 다른 레이아웃을 쓰라고 강제하는 게 맞을지도 모르겠다.

## 남은 것

`layout-*` 클래스가 잔존한 상태로 AI가 스타일을 덧씌우면 shell CSS가 scoped CSS를 이기는 문제를 한 번 겪었다. specificity 충돌이었다. 지금은 정규화로 우회 중인데, 근본적으로는 shell CSS를 `:not()`으로 가드하는 게 맞다. 다음 라운드에 손볼 부분이다.

레이아웃을 하나 더 추가하고 싶은 후보가 두 개 남아있다 — timeline과 quote-focus. 이건 실제로 필요한 슬라이드가 나올 때까지 미룰 생각이다. 만들어놓고 안 쓰는 패턴이 늘어나는 게 제일 관리하기 싫다.