---
title: "PPT 도식을 코드로 다루기: SVG 색상 변환부터 발표용 PNG 생성까지"
date: 2026-07-20 17:00:00 +0900
categories: [Tooling]
tags: [svg, python, presentation]
---

발표 전날 밤에 "색상 톤 좀 맞춰줄래요?"가 온다. 예전 같으면 PPT를 열고 도형을 하나씩 클릭했다. 지금은 터미널을 연다.

## PPT를 그만두게 된 순간

같은 스타일의 도식이 슬라이드 다섯 장에 흩어져 있었다. 브랜드 컬러가 바뀌었다는 통보를 받은 게 발표 전날 저녁. 도형 하나에 채우기 색, 테두리 색, 그림자 색까지 세 번씩 클릭. 다섯 장이면 그것만 몇 십 번이다. 중간에 딴 도형을 잘못 눌러서 위치를 미세하게 어긋뜨리기도 했다. 그러면 그걸 다시 맞추느라 또 시간이 간다.

그날 이후로 도식을 SVG로 관리하기 시작했다. SVG는 결국 텍스트다. 색상도 텍스트, 좌표도 텍스트. 그러면 `sed` 든 파이썬이든 뭐로 열든 일괄 수정이 된다.

```xml
<rect x="40" y="40" width="200" height="80"
      fill="{color_primary}" stroke="{color_text}" />
<text x="140" y="90" fill="{color_text}">파이프라인</text>
```

색이 들어갈 자리를 `{color_primary}` 같은 자리표시자(placeholder, 나중에 채울 빈칸)로 남겨둔다. 이 자리표시자가 이 워크플로우의 전부다. 나머지는 이걸 뭐로 채우느냐의 문제일 뿐이다.

## 팔레트를 딕셔너리로

한 발표 자료에 팔레트가 보통 두세 개 필요하다. 무대에서 쓸 브랜드 컬러 버전, 흑백 프린트용, 어두운 배경 슬라이드용 화이트 버전. 이걸 딕셔너리로 두면 파일별로 색을 새로 정할 필요가 없다.

```python
PALETTES = {
    "brand": {
        "primary":   "#2E74B5",
        "secondary": "#D6E4F0",
        "text":      "#1F2D3D",
    },
    "mono": {
        "primary":   "#333333",
        "secondary": "#EEEEEE",
        "text":      "#000000",
    },
    "dark_bg": {
        "primary":   "#FFFFFF",
        "secondary": "#B7C7D9",
        "text":      "#FFFFFF",
    },
}

def apply_palette(svg: str, palette: dict) -> str:
    out = svg
    for key, color in palette.items():
        out = out.replace(f"{{color_{key}}}", color)
    return out
```

문자열 치환이 그렇게 무식해 보이지만, SVG의 자리표시자 개수가 도식 하나에 스물 몇 개를 넘지 않아 이걸로 충분하다. XML 파서를 쓸까 잠깐 고민했는데, 파서를 태우면 속성 순서가 바뀌거나 공백이 재정렬되면서 git diff가 지저분해진다. 그게 싫어서 그냥 문자열로 갔다.

## 제목은 슬라이드에게, 그림은 도식에게

한동안은 도식 안에 제목을 박아 넣었다. "메타 파이프라인" 같은 문구가 도식 상단에 붙어 있는 식. 그런데 같은 도식을 다른 슬라이드에서 재활용하려니 제목만 다르게 하고 싶어졌다. 결국 도식 파일을 복사해서 제목만 바꾼 사본을 만들었다. 사본이 세 개쯤 생기고 나서 이건 아니라고 판단했다.

그 뒤로는 SVG에서 제목 영역에 `id="title-layer"` 를 달아둔다. 슬라이드에 넣을 때는 이 레이어를 벗겨낸 버전을 쓴다. 제목은 슬라이드의 텍스트 상자로 따로 얹는다.

```python
from lxml import etree

NS = {"svg": "http://www.w3.org/2000/svg"}

def strip_title_layer(svg_path: str, out_path: str) -> None:
    tree = etree.parse(svg_path)
    root = tree.getroot()
    for el in root.findall(".//*[@id='title-layer']", NS):
        el.getparent().remove(el)
    tree.write(out_path)
```

원본은 하나, 산출물은 여러 개. 이 원칙만 지키면 도식이 늘어나도 관리 부담이 곱셈이 아니라 덧셈으로 늘어난다.

## PNG로 떨어뜨리는 건 결국 rsvg-convert

슬라이드 도구에 SVG를 직접 넣을 수 있을 때도 있지만, 협업 상대가 어떤 도구를 쓸지 모른다. 결국 PNG가 가장 안전하다. `rsvg-convert`(librsvg 기반 CLI)를 쓴다. 한글 폰트 렌더링이 깔끔하고 배치도 빠르다.

```bash
rsvg-convert --dpi-x 192 --dpi-y 192 diagram.svg -o diagram.png
rsvg-convert --width 1920 diagram.svg -o diagram_wide.png
```

일괄 변환은 파이썬으로 감싼다.

```python
import subprocess
from pathlib import Path

def render_all(src_dir: str, dst_dir: str, width: int = 1920) -> None:
    Path(dst_dir).mkdir(parents=True, exist_ok=True)
    for svg in Path(src_dir).glob("*.svg"):
        out = Path(dst_dir) / f"{svg.stem}.png"
        subprocess.run(
            ["rsvg-convert", "--width", str(width), str(svg), "-o", str(out)],
            check=True,
        )
```

`check=True` 를 붙여두면 폰트를 못 찾아서 조용히 빈칸으로 렌더된 파일을 밤늦게 발견하는 사고를 막을 수 있다. 예전에 한 번 당했다.

## 도식과 대본이 어긋나는 문제

한번은 리허설에서 "지금 슬라이드는 정밀도 0.82라고 말하고 있는데, 도식은 0.7x로 남아 있네요"라는 말을 들었다. 실험 결과가 갱신됐는데 도식은 예전 값이었다. 대본만 고치고 도식은 잊었던 것.

그 뒤로 수치도 팔레트처럼 자리표시자로 뺐다. 스크립트가 실험 결과를 읽어서 도식에 박아 넣는다. 도식과 대본이 같은 소스에서 나오게 되면, 최소한 두 값이 어긋날 일은 없다.

```python
METRICS = {"precision": 0.82, "recall": 0.76, "f1": 0.79}

svg = Path("templates/metrics.svg").read_text()
for k, v in METRICS.items():
    svg = svg.replace(f"{{{k}}}", f"{v:.2f}")
svg = apply_palette(svg, PALETTES["brand"])
Path("out/metrics_brand.svg").write_text(svg)
```

이 방식이 매끈하지는 않다. 자리표시자 이름이 겹치면 조용히 오작동하고, 소수점 표기를 소스마다 통일하지 않으면 도식에만 자릿수가 다르게 찍힌다. 지금은 실험 결과가 담긴 파일과 도식 스크립트를 같은 폴더에 두는 정도로 대응하고 있는데, 이걸 좀 더 정돈된 구조로 바꾸는 건 아직 못 했다. 발표가 급할 때는 결국 손으로 한 번 훑는다. 그게 지금의 한계다.