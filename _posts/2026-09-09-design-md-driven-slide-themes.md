---
title: "디자인 시스템을 Markdown으로 모아두고 발표자료 테마로 쓰기"
date: 2026-09-09 17:00:00 +0900
categories: [Frontend]
tags: [marp, design-system, markdown, presentation, css]
---

발표자료를 만들 때마다 같은 일을 하고 있었다. "브랜드 컬러는 #1A2B3C고, 폰트는 Pretendard, 버튼 모서리는 8px…" 이걸 매번 프롬프트에 써넣었다. 그러다 문득 이상하다는 생각이 들었다. 내가 지금 디자인을 하는 건지, 디자인을 *설명*하는 건지.

## 문제는 디자인 정보가 텍스트로 흩어져 있다는 것이었다

Marp(마크다운을 슬라이드로 변환하는 도구)로 발표자료 생성 흐름을 만들면서, 디자인 선택이 발목을 잡았다. 내용은 구조화할 수 있는데 스타일을 어떻게 주입할지가 계속 즉흥이었다. AI가 CSS를 매번 새로 쓰면 결과물이 일관되지 않는다. 그 전에 정해진 디자인 규칙이 있어야 했다.

방향을 바꿨다. 디자인 명세를 파일로 분리하고, 발표자료를 만들 때는 그 파일에서 읽어오는 구조로.

## `DESIGN.md`를 디자인 데이터베이스로

브랜드별로 `DESIGN.md` 파일을 하나씩 만들었다. 경로는 `00_System/design_styles/<brand>/DESIGN.md` 형태다. 파일 안에는 색상 팔레트, 타이포그래피 스케일, spacing 단위, 컴포넌트 규칙이 들어간다. Figma 디자인 토큰을 옮겨 적는 것과 비슷하다. 차이는 이게 Markdown이라 읽기도 쉽고 버전 관리도 되고, 그대로 컨텍스트로 넣을 수 있다는 것이다.

한 파일에 이런 식으로 담는다:

```markdown
## Colors
- primary: #1A2B3C
- accent: #F5A623
- surface: #FFFFFF
- text-main: #222222

## Typography
- base: Pretendard, 16px
- heading-1: 2rem / 700
- heading-2: 1.5rem / 600

## Spacing
- unit: 8px
- section-gap: 48px

## Components
- card: border-radius 12px, shadow light
- button: radius 8px, padding 12px 24px
```

이게 있으면 "이 디자인 시스템을 기반으로 Marp 테마를 작성해"라고 하면 된다. 매번 설명하지 않아도 된다.

## 문제: 모든 파일에 frontmatter가 있지는 않았다

디자인 파일을 모을 때 골치 아픈 케이스가 있었다. 일부 파일은 YAML frontmatter로 메타데이터가 정리돼 있었고, 일부는 그냥 Markdown 본문만 있었다. 파일 이름이나 첫 번째 heading에서 이름을 유추해야 했다.

그래서 파서를 두 단계로 짰다. 먼저 `_parseFrontmatterMeta()`로 YAML 블록이 있는지 확인하고, 없으면 `_parseMarkdownFallback()`으로 넘어간다. fallback에서는 첫 번째 `# heading`을 이름으로, 첫 문단을 설명으로, `##` 이하에서 색상 hex값 패턴을 찾아 swatch 배열을 만든다.

코드를 직접 싣지 않는 이유가 있다. 이 파서는 로컬 하네스 안에 있어서 지금 여기서 꺼낼 수가 없다. 구조만 적는다: fallback은 정규식으로 hex 색상(`#[0-9A-Fa-f]{3,6}`)을 뽑고, 각 색상 옆에 붙은 한 줄 코멘트를 label로 쓴다. frontmatter가 없어도 색상 팔레트 정도는 자동으로 읽힌다.

## 사용자는 내용을 쓰고, 디자인은 고른다

이 구조의 진짜 의도는 분리다.

발표자료를 만드는 흐름이 이렇게 됐다:

1. 사용자가 내용(본문 Markdown)을 넣는다
2. 수집된 DESIGN.md 파일 목록을 카드 UI로 보여준다 — 이름, 설명, 색상 swatch 미리보기
3. 스타일을 고르면 해당 디자인 명세를 읽어 Marp CSS 테마로 변환한다
4. 내용 + 테마를 합쳐 PPTX와 HTML 두 포맷으로 동시에 뽑는다

카드 UI에서 고르는 단계가 중요하다. 이 단계가 없으면 AI가 "적당히 예쁘게" 만드는 결정을 혼자 내린다. 그 결과물은 한 번은 괜찮아도 다음번에 다르게 나온다. 선택권을 사용자에게 주면 결과의 분산이 줄어든다.

## 반응형 레이아웃도 선택지가 된다

HTML 출력은 PPTX와 달리 화면 크기에 따라 레이아웃이 달라질 수 있다. 이걸 그냥 두면 모바일에서 깨진다. 그래서 레이아웃 프리셋도 DESIGN.md에 선택적으로 담았다: 단일 컬럼, 2단 그리드, 타이틀+본문 스택 같은 식으로. HTML 렌더링 시에 이 프리셋을 읽어 CSS grid 설정을 바꾼다.

아직 해결 못 한 게 있다. PPTX 출력에서 한국어 폰트가 임베드가 안 되는 케이스가 있다. Marp CLI가 내부적으로 Chromium을 쓰는데(Puppeteer 기반), 시스템에 해당 폰트가 없으면 fallback 폰트로 바뀐다. DESIGN.md에 `font-family`를 명시해도 환경에 따라 다르게 나온다. 배포 환경에 폰트를 미리 설치해두는 방법과, woff2를 CSS에 직접 embed하는 방법 중 어느 쪽이 맞는지 아직 결론을 못 냈다.