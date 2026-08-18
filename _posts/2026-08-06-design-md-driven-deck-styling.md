---
title: "발표자료 디자인을 코드가 아니라 design.md로 고르는 실험"
date: 2026-08-06 17:00:00 +0900
categories: [Frontend]
tags: [design-system, markdown, tooling]
---

발표자료 생성기를 만들다가 이상한 지점에서 막혔다. 콘텐츠는 Markdown으로 받아오는 게 자연스러운데, 정작 "어떤 톤으로 보일지"는 코드 어딘가 하드코딩된 hex 문자열과 폰트 이름으로 흩어져 있었다. 브랜드가 하나 늘 때마다 조건 분기가 하나 더 붙었다.

## 처음엔 그냥 드롭다운이면 된다고 생각했다

테마 세 개, 네 개까지는 그게 맞았다. 문제는 다섯 번째부터였다. 색만 바뀌는 게 아니라 폰트가 바뀌고, 카드 모서리 곡률이 바뀌고, 여백 밀도가 바뀌었다. 어떤 브랜드는 카드 컴포넌트를 안 쓰고, 어떤 브랜드는 badge를 쓴다. "테마"라는 게 hex 세 개로 정리되지 않았다.

그래서 방향을 뒤집었다. 디자인을 코드에서 꺼내서 파일 하나로 만들자. 파일 이름은 `design.md`. 하나의 파일이 하나의 브랜드다.

## design.md 하나가 담는 것

frontmatter — 문서 머리에 `---` 로 감싸는 YAML 메타 블록 — 로 다 밀어넣었다.

```yaml
---
name: "Codnut Dark"
description: "진한 배경 위에 형광 포인트 색상 조합"
colors:
  primary: "#7C3AED"
  background: "#0F0F0F"
  text: "#F3F4F6"
typography:
  heading: "Pretendard"
  body: "Pretendard"
  size_base: 16
rounded: "0.75rem"
spacing: "relaxed"
components:
  - card
  - badge
  - divider
---
```

밑에 본문에는 이 브랜드가 어떤 인상을 노리는지, 어떤 상황에 안 어울리는지를 자연어로 적었다. 이건 사람이 읽으라고 쓴 거지 파서가 읽는 부분은 아니다.

디렉터리에 이 파일들만 쌓인다. 코드는 디렉터리를 스캔해서 카드 목록을 만든다. 새 테마를 만들려면 코드가 아니라 파일을 하나 더 넣으면 된다.

## 파서는 두 갈래로 갈렸다

이상적으로는 위처럼 frontmatter가 있어야 한다. 그런데 실제로 쌓여 있던 예전 문서들은 그냥 마크다운이었다. H1 하나, 설명 한 문단, 본문 어딘가에 색상 hex가 박혀 있는 형태.

이걸 버릴까 잠깐 고민했는데, 버리면 재사용 후보를 통째로 놓친다. 그래서 fallback을 만들었다.

```python
def _parseFrontmatterMeta(content: str) -> dict:
    match = re.match(r'^---\n(.*?)\n---', content, re.DOTALL)
    if not match:
        return {}
    return yaml.safe_load(match.group(1)) or {}
```

frontmatter가 없으면 이쪽으로 빠진다.

```python
def _parseMarkdownFallback(content: str) -> dict:
    name = re.search(r'^# (.+)', content, re.MULTILINE)
    desc = re.search(r'^(?!#)[^\n]+', content.split('\n\n', 1)[-1])
    colors = re.findall(r'#[0-9A-Fa-f]{6}', content)
    return {
        "name": name.group(1) if name else "Unnamed",
        "description": desc.group(0) if desc else "",
        "colors": {
            "primary": colors[0] if len(colors) > 0 else "#333",
            "secondary": colors[1] if len(colors) > 1 else "#666",
            "background": colors[2] if len(colors) > 2 else "#fff",
        }
    }
```

H1을 테마 이름으로, 첫 단락을 설명으로, 본문에 나오는 hex를 등장 순서대로 primary/secondary/background에 꽂는다. 완벽하지 않다. 문서 안에서 예시로만 등장한 색이 primary로 잡히는 사고도 몇 번 났다. 그래도 최소한 카드 하나는 만들어졌고, 잘못 잡혔으면 frontmatter를 붙여서 정답을 알려주면 된다. 일부러 그렇게 방향을 잡았다 — 자동 추론은 "일단 뭐라도 보여준다" 까지만.

## 파싱한 뒤가 사실 더 애매하다

frontmatter에서 나온 값과 fallback에서 나온 값은 필드 개수부터 다르다. 앞은 typography까지 있고 뒤는 색 세 개가 전부다. 그래서 파싱 직후에 한 번 정규화 단계를 두고, 없는 필드는 프로젝트 기본값으로 채운 다음에야 렌더링 쪽으로 넘긴다.

이 정규화 층이 없으면 렌더링 코드가 `if meta.get('typography')` 로 뒤덮인다. 한번 그렇게 짜봤다가 되돌렸다. 렌더러가 "필드는 항상 있다" 를 전제로 짜여 있어야 다른 사람이 읽을 만한 코드가 된다.

## 코드-디자인 분리가 실제로 준 것

디자인 파일을 고치는 사람과 코드를 고치는 사람이 갈렸다. 색을 하나 바꾸는데 배포 파이프라인을 타지 않아도 됐다. 이게 생각보다 큰 변화였다. "색 좀 바꿔주세요" 가 티켓이 되지 않았다.

## 아직 안 풀린 것

fallback 파서가 색을 잘못 잡는 케이스는 여전히 있다. 문서 안 표에 예시로 들어간 hex를 primary로 오인하는 실수. H2 아래 첫 hex만 본다든지, 코드블록 안은 무시한다든지, 규칙을 더 조일 수는 있는데 조일수록 fallback이 fallback답지 않아진다. 원래 이건 "이 문서 대충 파싱해서 카드 하나만 걸어둬" 를 위한 층이지 정확성을 위한 층이 아니다.

한 번 더 고민 중인 건 design.md에 담지 못하는 것들이다. 애니메이션 타이밍이나 리스트 마커 모양 같은 건 여전히 CSS 파일에 남아있다. 이걸 마크다운으로 밀어넣는 건 무리라고 생각하는데, 그럼 결국 "코드에 남긴 부분" 과 "파일로 뺀 부분" 의 경계를 문서화해야 한다. 아직 안 썼다.