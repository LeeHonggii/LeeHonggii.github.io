---
title: "디자인 시스템을 Markdown으로 모아두고 발표자료 생성기에 연결하기"
date: 2026-08-12 17:00:00 +0900
categories: [Frontend]
tags: [design-system, markdown, pptx, ai-pipeline]
---

AI로 슬라이드를 자동 생성하는 파이프라인을 굴리다 보니, 어느 순간부터 내용보다 디자인이 더 큰 문제였다. LLM이 뽑아주는 문장은 어지간하면 쓸 만하다. 그런데 슬라이드를 열어보면 한 덱 안에서도 색이 매번 다르고, 어떤 슬라이드는 제목이 지나치게 크고, 다음 슬라이드는 여백이 없다.

처음엔 프롬프트로 해결하려고 했다. "메인 컬러는 #1A56DB로", "제목은 28pt로", "카드형 레이아웃으로". 프롬프트가 계속 길어졌다. 브랜드가 늘어나면 프롬프트를 브랜드 개수만큼 복제해야 했다. 어떤 지시는 지켜지고 어떤 지시는 조용히 무시됐다. 프롬프트가 스타일 정의를 겸하는 순간 유지보수 대상이 두 개로 쪼개진다는 걸 뒤늦게 알았다.

## 스타일을 프롬프트 밖으로 꺼내기

방향을 바꿨다. 스타일은 프롬프트가 아니라 파일에 적는다. 생성기는 그 파일을 읽어서 적용한다. LLM은 내용만 만든다.

브랜드별로 폴더를 하나씩 두고, 그 안에 `DESIGN.md`를 하나 넣는다.

```
design_styles/
  corporate-blue/DESIGN.md
  startup-dark/DESIGN.md
  minimal-white/DESIGN.md
```

`DESIGN.md`는 자연어와 key-value가 섞인 형태다. 사람이 열어서 고칠 걸 전제로 하기 때문에 JSON이나 YAML은 피했다. 콜론 뒤에 hex 하나, 그 정도면 파서가 알아본다.

```markdown
## Colors
- primary: #1A56DB
- background: #FFFFFF
- accent: #F59E0B

## Typography
- heading: Pretendard Bold, 32px
- body: Pretendard Regular, 16px, line-height 1.6

## Components
- card: border-radius 12px, shadow md
- code-block: bg #1E293B, fg #E2E8F0
```

이 파일이 단일 출처(single source of truth, 스타일 결정이 딱 한 군데에만 존재한다는 뜻)다. 프롬프트를 고칠 필요 없이 여기만 고치면 그 다음 생성물부터 반영된다.

## 디자인과 레이아웃은 다른 축이다

한동안은 색상·폰트와 슬라이드 구조를 한 파일에 뒤섞어 뒀다. 그러다 "bento grid(바둑판처럼 카드를 격자로 배치하는 레이아웃) 구조에 corporate-blue 브랜드를 씌우고 싶다"는 요구가 들어왔을 때 막혔다. 같은 레이아웃에 다른 브랜드, 같은 브랜드에 다른 레이아웃 — 이걸 조합해서 못 쓰면 파일이 브랜드×레이아웃 개수만큼 폭발한다.

축을 분리했다.

```
deck_layouts/
  bento-grid/LAYOUT.md
  magazine-editorial/LAYOUT.md
  code-focus/LAYOUT.md
  timeline/LAYOUT.md
```

레이아웃은 브랜드를 모르고, 브랜드는 레이아웃을 모른다. 서로 참조하지 않는다.

| 슬러그 | 언제 쓰나 |
|---|---|
| `bento-grid` | 비교·나열 |
| `magazine-editorial` | 표지, 스토리 시작 |
| `code-focus` | 기술 설명 |
| `timeline` | 회고, 로드맵 |

CLI는 두 축을 독립 인자로 받는다.

```bash
python make_deck.py input.md \
  --design corporate-blue \
  --layout bento-grid
```

슬라이드별로 레이아웃을 바꾸고 싶은 경우가 실제로 자주 생겼다. 표지는 editorial, 중간 코드 설명은 code-focus. 그래서 인덱스별 오버라이드를 하나 더 뒀다.

```bash
python make_deck.py input.md \
  --design startup-dark \
  --per-slide-layouts '{"0": "magazine-editorial", "3": "code-focus"}'
```

이 부분은 초반에 넣을지 말지 고민을 좀 했다. 사람이 JSON을 CLI에 직접 쓰는 건 지저분하다. 다만 대안(별도 설정 파일)은 파일 수를 더 늘리기 때문에, 지금 단계에선 CLI에 남겨뒀다. 슬라이드 수가 20을 넘어가면 이 방식이 버거워질 것 같은데, 아직 거기까지 안 갔다.

## PPTX와 HTML을 한 소스에서

출력은 두 갈래다. 편집 가능한 PPTX 하나, 웹에서 바로 볼 수 있는 반응형 HTML 하나. 둘을 별도 파이프라인으로 관리하지 않는 게 이 구조의 값어치다. 둘 다 같은 `DESIGN.md`와 `LAYOUT.md`를 읽는다.

- PPTX 쪽은 색상을 `RGBColor`, 크기를 `Pt()`로 바인딩한다.
- HTML 쪽은 CSS 커스텀 프로퍼티(`--primary`, `--heading-size`)로 주입하고, 반응형은 레이아웃 템플릿의 Grid/Flexbox가 감당한다.

바인딩 로직은 포맷별로 나뉘지만, 결정은 한 군데에서만 내려진다. PPTX와 HTML이 서로 다른 색으로 나올 일이 구조적으로 사라진다.

## 아직 못 푼 것

이 구조로 넘어오고 나서 브랜드 추가는 확실히 편해졌다. 폴더 하나 만들고 `DESIGN.md` 한 장 채우면 그날부터 새 브랜드로 뽑힌다.

그렇다고 다 잘 굴러가는 건 아니다. `DESIGN.md`에 accent 색을 하나 추가하는 순간 LLM이 그걸 주 팔레트로 오해해서 슬라이드 전체 톤을 accent 색으로 물들여버린 적이 있다. 정의 시점에 모아둔 정보를, 생성 시점의 모델이 "얼마나 강하게 참고할지"는 여전히 통제 밖이다. 화이트리스트로 hex 허용 목록을 걸어보고 있는데, 이게 근본 해결인지 확신은 없다. accent를 아예 별도 섹션으로 빼는 것도 생각 중이다.

`LAYOUT.md`도 지금은 텍스트 설명 위주라서, 레이아웃끼리의 시각적 일관성이 사람 눈에만 검증된다. 렌더링을 캡처해서 자동으로 비교하는 루프를 붙일지, 아니면 그냥 이대로 둘지 — 아직 고민 중이다.