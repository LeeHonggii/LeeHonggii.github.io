---
title: "Markdown 하나로 PPTX와 HTML 만들기: ‘내용 유실 없음’은 어떻게 검증할까?"
date: 2026-10-05 17:00:00 +0900
categories: [Tooling]
tags: [python-pptx, Marp, Markdown, 발표자료자동화, 정규식]
---

발표자료를 만들 때마다 같은 스크립트를 새로 짜고 있었다. 이 글은 그 스크립트를 Markdown 한 장으로 대체한 과정이다. 마지막에 PPTX가 안 나오는 문제가 남아 있어서, 끝도 깔끔하지 않다.

## 슬라이드마다 좌표를 박던 시절

처음엔 python-pptx로 슬라이드를 직접 만들었다. 발표 한 건당 스크립트가 하나였다. 표 헬퍼는 이런 모양이다. 식별자와 색은 일반화했다.

```python
def table(slide, x, y, w, rows, col_w=None, fs=11):
    nr, nc = len(rows), len(rows[0])
    h = Inches(0.32 * nr)
    gt = slide.shapes.add_table(nr, nc, x, y, w, h).table
    if col_w:
        for j, cw in enumerate(col_w):
            gt.columns[j].width = cw
    for i, row in enumerate(rows):
        for j, val in enumerate(row):
            c = gt.cell(i, j); c.margin_top = Pt(2); c.margin_bottom = Pt(2)
            c.margin_left = Pt(6)
            tf = c.text_frame; tf.word_wrap = True
            p = tf.paragraphs[0]; p.text = str(val)
            p.font.size = Pt(fs)
            if i == 0:
                c.fill.solid(); c.fill.fore_color.rgb = HEAD
                p.font.bold = True; p.font.color.rgb = WHITE
            else:
                c.fill.solid(); c.fill.fore_color.rgb = WHITE
                p.font.color.rgb = DARK
    return gt
```

헬퍼는 괜찮았다. 문제는 호출하는 쪽이었다.

```python
section(s, Inches(0.4), Inches(3.25), Inches(6), '비교 관점')
table(s, Inches(0.4), Inches(3.65), Inches(7.0), [
    ['관점', '비교 기준', '출처'],
    # ...
], col_w=[Inches(1.3), Inches(4.5), Inches(1.2)], fs=11)

section(s, Inches(7.8), Inches(3.25), Inches(5), '정렬 결과')
table(s, Inches(7.8), Inches(3.65), Inches(5.1), [ ... ],
      col_w=[Inches(1.3), Inches(3.8)], fs=12)
```

모든 요소의 x, y, 폭이 인치 단위 숫자로 들어 있다. 행이 하나 늘면 아래 요소의 y를 전부 손으로 밀어야 한다. 글자 크기도 칸마다 11, 12, 10으로 눈대중이다. 내용은 사실 Markdown 표 하나로 끝나는 정보였다. 레이아웃 계산이 내용보다 코드를 더 많이 차지하고 있었다.

두 번째 스크립트는 기존 회사 템플릿 위에 얹는 방식이었다. 이쪽이 더 지저분했다.

```python
prs = Presentation(TEMPLATE)

_sldIdLst = prs.slides._sldIdLst
for sld in list(_sldIdLst):
    prs.part.drop_rel(sld.rId)
    _sldIdLst.remove(sld)

for _m in prs.slide_masters:
    for _sh in list(_m.shapes):
        if _sh.left is not None and _sh.left >= prs.slide_width:
            _sh._element.getparent().remove(_sh._element)
    for _lay in _m.slide_layouts:
        for _sh in list(_lay.shapes):
            if _sh.left is not None and _sh.left >= prs.slide_width:
                _sh._element.getparent().remove(_sh._element)
```

기존 슬라이드를 `drop_rel`로 떼어내고, 슬라이드 밖(x가 폭 이상)으로 밀려난 도형을 마스터와 레이아웃에서 지운다. 이런 코드는 템플릿이 바뀌면 바로 깨진다. 슬라이드 폭 밖에 도형이 숨어 있다는 걸 알아야만 쓸 수 있는 코드이기도 하다. 다시 쓸 수 있는 자산이 아니라 일회용 정비 작업에 가까웠다.

## 바꾼 구조

Markdown에서 시작하는 변환기 `make_deck.py`를 만들었다. 이 도구의 소스는 이번 글에 싣지 않는다. 흐름만 적는다. Python이 Markdown을 읽고 디자인 메타데이터를 입힌 뒤 Marp 형식의 Markdown을 만든다. 거기서 Marp CLI가 두 형식을 뽑는다.

```bash
make_deck.py <md>
marp deck.md --pptx
marp deck.md --html
```

같은 원본에서 편집 가능한 PPTX와 HTML이 나온다. 한 번 만든 내용으로 두 가지 배포 형태를 얻는 게 목적이었다.

옵션은 두 갈래다.

```bash
--layout <slug>
--per-slide-layouts JSON
```

`--layout`은 덱 전체에 쓰는 레이아웃이다. `--per-slide-layouts`는 슬라이드별 지정이다. 지정된 슬라이드에는 `<!-- _class: layout-<slug> -->` 지시자가 붙는다. Marp가 이 클래스를 CSS로 받는다.

## 브랜드와 레이아웃을 갈라놓은 이유

처음엔 "스타일" 하나로 뭉뚱그렸다. 색·글꼴과 페이지 구도를 한 덩어리로 다루니 조합이 터졌다. 브랜드 A에 레이아웃 3종, 브랜드 B에 같은 3종이면 템플릿이 6개가 된다.

그래서 둘을 분리했다. 브랜드(색상·타이포그래피)는 덱 전체에 하나만 적용한다. 레이아웃은 슬라이드마다 고른다. UI는 디자인 카드와 레이아웃 탭 두 군데로 나눴다. 표지는 큰 제목 레이아웃, 비교 슬라이드는 2단 레이아웃을 쓰면서도 브랜드는 안 바뀐다.

## 내용 유실 없음: 방침과 검증

변환기에 세운 원칙은 하나다. 내용이 슬라이드에 넘치면 삭제하지 말고 슬라이드를 나눈다. 요약해서 줄이는 편이 보기엔 깔끔하겠지만, 원본 문장이 사라지면 변환기를 믿을 수 없게 된다.

검증은 이렇게 했다. 문장, 코드 블록, 표를 원본과 출력에서 각각 대조했다. 그리고 스모크 테스트(최소한의 동작 확인)를 돌렸다. 이 정도다.

이것으로 "모든 입력에서 무손실"을 증명했다고는 말 못 한다. 대조한 입력이 한정돼 있고, 슬라이드를 나누는 경계에서 코드 블록이나 표가 쪼개지는 경우는 체계적으로 시험하지 않았다. 표의 병합 셀처럼 아직 보지 못한 입력도 있다. "유실 없음"이라는 제목의 질문에 지금 내가 줄 수 있는 답은 "내가 넣어본 입력에서는 확인했다"까지다.

## 정규식이 멈췄다

디자인 문서(브랜드 정의 문서)를 파싱하는 부분에서 처리가 사실상 멈췄다. 정규식 하나가 catastrophic backtracking(여러 경로를 되짚다 실행 시간이 폭발하는 현상)에 빠진 것이다. 입력이 조금만 길어져도 CPU가 계속 돌았다.

고친 방법은 정규식을 다듬는 게 아니었다. 라인 기반 파서로 바꿨다. 줄을 하나씩 읽고, 지금 어느 구간에 있는지 상태만 들고 간다. 복잡한 패턴 하나로 문서 전체를 먹으려 한 게 잘못이었다.

이어서 두 가지를 보완했다.

- YAML에서 `description: |` 같은 블록 스칼라(여러 줄 값)를 제대로 읽도록 했다.
- frontmatter 없는 Markdown 문서에서도 제목·설명·색상을 뽑는다. UI 쪽에는 `_parseFrontmatterMeta`와 frontmatter가 없을 때 쓰는 `_parseMarkdownFallback`을 뒀다.

정규식을 라인 파서로 바꾼 뒤 속도를 따로 재진 않았다. "멈추던 입력이 안 멈춘다" 정도만 확인했다.

## 돌려본 결과

5개 슬라이드짜리 입력으로 PPTX와 HTML이 둘 다 나오는 걸 확인했다. 여기까지는 좋았다.

그런데 마지막에 문제가 하나 들어왔다. UI의 생성 버튼을 눌러도 PPTX가 나오지 않는다는 것이다. 내 환경에서는 같은 입력이 나왔는데 사용자 쪽에서는 안 나온다.

지금 의심하는 건 이렇다. 변환 도중 오류가 나는데, 그 메시지가 stderr에만 찍히고 UI에는 전달되지 않는다. 그래서 사용자 눈에는 "버튼을 눌렀는데 아무 일도 없다"로 보인다. 다만 이건 추정이다. 사용자 환경의 stderr를 실제로 받아보지 못했고, Marp 실행 파일 경로나 브라우저 의존성 문제 같은 다른 원인도 배제하지 못했다. 해결됐는지는 이 글을 쓰는 시점에 확인하지 못했다.

HTML이 반응형이라고 만들었지만 모바일에서 실제로 어떻게 보이는지는 아직 확인하지 않았다. CSS상 그렇게 작성했다는 것뿐이다.

다음에 할 일은 정해져 있다. stderr를 UI까지 올려서 실패가 눈에 보이게 만들고, 그 메시지를 받은 다음에야 PPTX 문제의 원인을 말할 수 있다.