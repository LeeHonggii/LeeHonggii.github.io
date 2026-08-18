---
title: "Word 보고서에 Mermaid를 넣으려다 알게 된 편집 가능성과 안정성의 trade-off"
date: 2026-07-22 17:00:00 +0900
categories: [Tooling]
tags: [docx, mermaid, pandoc, ooxml, automation]
---

발단은 요건 한 줄이었다. 어떤 백엔드 서비스 구조를 비개발자용 보고서로 정리해야 했고, 이미 Markdown → pandoc → DOCX 파이프라인은 굴러가고 있었다.

```bash
pandoc 시스템_설명서.md -o 시스템_설명서.docx
```

여기까진 무난했다. 걸린 건 다이어그램이었다. Mermaid(머메이드 — 텍스트로 다이어그램을 그리는 문법)로 짠 플로우차트를 Word에 넣되, **담당자가 그 다이어그램을 직접 손보고 싶다**는 말이 붙었다. PNG는 클릭해도 안 열리니까 안 된다는 얘기다.

## "그냥 도형으로 그리면 되잖아" 라고 생각했다

python-docx(파이썬으로 DOCX를 만드는 라이브러리)부터 뒤졌다. 표, 이미지, 스타일까지는 다 되는데, `add_shape` 비슷한 게 없다. 도형 그리는 API가 아예 없었다. 이슈 트래커에도 "지원 계획 없음"으로 한참 전에 닫혀 있었다.

그럼 XML을 직접 쓰면 되지 않나 — 이게 두 번째 오판이었다.

## Raw OOXML을 직접 조립해 보다

OOXML(Office Open XML — Word 문서 내부 XML 포맷)에서 도형은 `<wps:wsp>`, 연결선은 `<wps:cxnSp>` 로 표현된다. 문단에 이걸 주입하면 이론상 "Word 네이티브 도형"이 된다. 박스 하나까진 실제로 나왔다. 문제는 두 개째부터였다.

좌표를 전부 손으로 계산해야 한다. 도형 위치는 EMU(English Metric Unit — 1인치 = 914,400) 단위인데, 노드 3개만 넘어가면 x·y 를 다 잡아줘야 한다. 화살표는 더 골치였다. Mermaid에서 `A --> B` 라고 쓰면 알아서 붙는 선이, 여기선 시작점·끝점 좌표를 각각 명시해야 하고, 박스 위치가 바뀌면 선도 다시 계산해야 한다. Mermaid가 대신 해주던 그래프 레이아웃 알고리즘을 내가 다시 짜야 한다는 소리다.

여기서 그만뒀어야 하는데, 하나 더 있었다. Word 2019와 Microsoft 365가 같은 XML을 다르게 렌더했다. 줄바꿈 위치가 미묘하게 다르거나, 도형이 살짝 겹치거나. 이건 코드에서 못 잡는다.

"편집은 되는데 안정성은 없는" 상태가 만들어졌다. 담당자한테 넘겨줄 수가 없는 결과물이다.

## 목표를 다시 정했다

한 발 물러섰다. 이 보고서에서 다이어그램은 **읽히기 위한 것**이지 편집되기 위한 게 아니다. 담당자가 실제로 다이어그램을 고칠 일이 얼마나 자주 있을까. 있다 해도, 그 수정 비용이 렌더링 깨짐을 디버깅하는 비용보다 크진 않을 것 같았다.

그래서 흐름을 이렇게 바꿨다. Markdown에서 Mermaid 코드블록만 뽑아내고, Playwright(브라우저 자동화)로 로컬 `mermaid.js` 를 돌려 이미지로 렌더한 다음, 그 이미지를 DOCX에 삽입한다.

```python
from playwright.sync_api import sync_playwright

def render_mermaid_to_png(mermaid_code: str, output_path: str):
    with sync_playwright() as p:
        browser = p.chromium.launch()
        page = browser.new_page()
        page.set_content(build_html(mermaid_code))
        page.locator(".mermaid svg").screenshot(path=output_path)
        browser.close()
```

이미지로 박아 넣으면 Word 버전이 뭐든 똑같이 보인다. 로컬 번들을 쓰니 CI에서도 인터넷 없이 돌아간다. 다이어그램이 바뀌면 Mermaid 소스만 고치고 다시 돌리면 끝이다.

편집 가능성을 완전히 버린 건 아니다. DOCX 부록에 Mermaid 소스를 그대로 텍스트로 붙여뒀다. 담당자가 수정이 필요하면 소스를 고쳐서 나한테 주고, 나는 재생성한다. 이 왕복이 XML을 손으로 고치는 것보다 압도적으로 빠르다.

## 남는 질문

편집 가능성은 기능이 아니라 유지 비용이었다. 이걸 처음부터 알았다면 OOXML 삽질은 안 했을 텐데, 아마 다음에 비슷한 요건이 오면 또 한 번은 흔들릴 것 같다. "네이티브로 만들 수 있으면 그게 최선 아닌가" 라는 감각이 은근히 세다.

아직 안 재본 게 하나 있다. 부록에 Mermaid 소스를 붙여둔 방식이 담당자한테 실제로 얼마나 쓰이는지, 아니면 그냥 이미지 상태로 두는지. 이게 정말로 "편집 가능성"의 대체재가 되는지는 다음 보고서 사이클을 한 번 더 돌려봐야 알 것 같다.