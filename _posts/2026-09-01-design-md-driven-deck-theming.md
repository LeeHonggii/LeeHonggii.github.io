---
title: "디자인 감각을 프롬프트가 아니라 design.md 파일로 주입하기"
date: 2026-09-01 17:00:00 +0900
categories: [Frontend]
tags: [design-system, markdown-driven, llm-tooling, presentation-deck, developer-tools]
---

발표자료 생성 도구를 만들다 보니 한 가지 패턴이 계속 반복됐다. Markdown 하나를 던지면 그럴듯한 슬라이드가 나오는데, 색상이 매번 달랐다. 같은 프롬프트를 써도 어떤 날은 파란 계열, 어떤 날은 초록. LLM이 그때그때 디자인을 새로 "발명"하고 있었다.

브랜드 스타일을 프롬프트 안에 직접 넣으면 어떨까 싶었다. 실제로 해봤다. "primary color는 #XXXXXX, heading font는 ○○"를 프롬프트 맨 앞에 붙이면 동작은 했다. 그런데 유지보수가 문제였다. 한 브랜드의 색상이 바뀌면 그 프롬프트가 박힌 코드를 찾아 고쳐야 했다. 브랜드가 두 개, 세 개로 늘어나면서 이 방식이 버텨낼 수 없다는 게 금방 보였다.

---

답은 파일이었다. 브랜드마다 `DESIGN.md` 하나를 두고, 도구가 슬라이드를 만들기 전에 그걸 읽도록 했다.

파일 경로는 `00_System/design_styles/<brand>/DESIGN.md` 형태로 고정했다. 어떤 브랜드인지는 UI에서 드롭다운으로 고르면 이 경로로 매핑된다. LLM이 디자인을 발명하는 게 아니라 이 파일에 적힌 스펙을 읽고 따르는 구조다.

파일 자체는 YAML frontmatter에 구조화된 스펙을 담는다.

```yaml
---
brand: "브랜드A"
colors:
  primary: "<hex>"
  secondary: "<hex>"
  accent: "<hex>"
  background: "<hex>"
  text: "<hex>"
typography:
  heading: "Pretendard"
  body: "Noto Sans KR"
  code: "JetBrains Mono"
  base_size: "16px"
spacing:
  unit: "8px"
  section_gap: "48px"
rounded: "12px"
components:
  card_shadow: "0 2px 8px rgba(0,0,0,0.08)"
  button_style: "filled"
---

이 브랜드는 신뢰감과 간결함을 강조한다.
색상은 차갑고 정돈된 느낌으로, 불필요한 장식 없이 정보가 앞에 서게 한다.
```

섹션이 `colors`, `typography`, `spacing`, `rounded`, `components` 다섯 개다. 처음에는 색상만 넣으려 했는데, 모서리 반경(`rounded`)이 다르면 같은 색을 써도 브랜드 느낌이 달라진다는 걸 실제로 써보고 알았다. 그래서 하나씩 추가했다.

---

분리해야 할 게 두 가지였다. **색상·브랜드**와 **레이아웃**이다.

처음엔 DESIGN.md에 레이아웃도 넣으려 했다. "2단 구성", "이미지 왼쪽" 같은 것들. 그런데 같은 브랜드라도 PPTX로 뽑을 때와 반응형 HTML로 뽑을 때 레이아웃이 달라야 했다. PPTX는 고정된 슬라이드 비율이고, HTML은 화면 너비에 따라 단 수가 바뀐다.

결국 브랜드 스펙과 레이아웃 템플릿을 별도 선택지로 뒀다. UI에서 "어떤 브랜드?"와 "어떤 레이아웃?"을 독립적으로 고른다. 두 선택이 합쳐져서 최종 산출물이 나온다. PPTX든 HTML이든 같은 DESIGN.md를 읽으니까 색상과 타이포그래피는 일관하게 유지된다.

---

예상치 못한 문제가 하나 더 있었다. UI에서 브랜드를 고를 때 카드 미리보기를 보여주는데, DESIGN.md가 없거나 frontmatter가 없는 파일이 들어오면 카드가 빈 상태로 렌더됐다.

frontmatter가 있으면 `_parseFrontmatterMeta()`가 brand 이름, 대표 색상, 설명을 뽑아낸다. 없으면 `_parseMarkdownFallback()`이 동작하는데 여기서 세 가지를 순서대로 시도한다. H1 태그에서 브랜드 이름을 가져오고, 첫 번째 본문 단락을 설명으로 쓰고, 파일 전체를 정규식으로 스캔해서 hex color 값을 추출한다. 그렇게 뽑은 색상을 카드 미리보기의 배경 계열로 쓴다.

완벽하진 않다. hex 값이 아예 없는 파일이면 기본 회색 배경으로 떨어진다. 그래도 "파일이 있음"과 "파일을 못 읽음"을 구분하는 것만으로도 사용자 경험이 달라졌다.

---

`components` 섹션이 아직 덜 됐다. 카드 그림자나 버튼 스타일 정도만 들어가 있고, 정작 슬라이드에 자주 쓰는 콜아웃 박스나 코드 블록 스타일은 반영이 안 된다. 어디까지 DESIGN.md에 넣고 어디서부터 레이아웃 템플릿이 책임질지 경계가 아직 흐릿하다. 지금은 일단 쓰면서 필요한 게 나올 때마다 추가하는 중이다.