---
title: "디자인 MD 71개를 읽다 깨진 YAML: frontmatter 없는 문서까지 처리한 파서 개선기"
date: 2026-08-13 17:00:00 +0900
categories: [Backend]
tags: [markdown, yaml, parser, pipeline]
---

## 반쯤 돌리고 나서야 알았다

디자인 시스템 스펙 문서를 자동으로 긁어와 내부 카탈로그에 얹는 파이프라인을 짜고 있었다. 소스는 공개된 디자인 시스템 저장소들. 파일은 전부 Markdown이고, 71개.

처음엔 별 생각 없었다. YAML frontmatter만 뽑아 쓰면 되니까. 렌더러가 title·description·색상만 잡아주면 끝날 일이었다.

절반쯤 처리 결과를 훑어보다가 이상함을 느꼈다. 카탈로그 카드에 title이 비어 있는 항목이 꽤 있었다. description은 더 심했다. 파서가 조용히 빈 객체를 돌려주고 있었던 것이다.

## 두 종류의 문서

파일을 하나씩 열어보니 원인은 단순했다. 포맷이 두 갈래로 갈려 있었다.

한쪽은 예상대로였다.

```markdown
---
title: "Button"
description: |
  인터랙션 가능한 기본 UI 요소.
  클릭, 탭, 키보드 입력을 모두 지원한다.
colors:
  - "#1A73E8"
  - "#FFFFFF"
---

## 사용법
...
```

다른 한쪽은 그냥 순수 Markdown이었다.

```markdown
# Card

카드 컴포넌트는 관련 정보를 하나의 묶음으로 보여준다.

색상: `#F5F5F5`, `#212121`
...
```

`---` 구분자를 기준으로 잘라내는 단일 파서를 쓰고 있었으니, 두 번째 종류는 통째로 무시되고 있었다. 문서 안에 정보는 다 있었다. 파서가 못 봤을 뿐이다.

기여자가 여럿인 오픈소스 문서에 "포맷 균일함"을 기대한 게 첫 번째 착각이었다.

## `description: |` 에서 한 번 더 깨졌다

frontmatter가 있는 쪽도 완벽하진 않았다. `description: |` 로 시작하는 여러 줄 서술이 문제였다. YAML의 block scalar — 파이프 뒤에 오는 여러 줄을 하나의 문자열로 이어붙이는 문법이다.

내가 처음 짜뒀던 건 `description:` 뒤를 정규식으로 잘라 오는 방식이었다. 그 결과가 이런 식으로 나왔다.

```
"description": "|\n  인터랙션 가능한 기본 UI 요소.\n  클릭, 탭, ..."
```

파이프 문자와 들여쓰기가 문자열에 그대로 눌러붙었다. 렌더러가 그걸 그대로 카드 설명에 뿌리고 있었다.

여기서 잠깐 regex를 더 다듬어볼까 싶었다. 그러다 관뒀다. block scalar, quoted string, flow scalar 다 스펙이 따로 있는데 그걸 손으로 흉내내는 건 나중에 반드시 어디선가 또 깨진다. `js-yaml` 같은 스펙 구현체에 맡기는 쪽이 맞다.

파서 진입점은 이렇게 정리됐다. frontmatter 블록만 잘라내서 라이브러리에 넘기고, 없거나 파싱이 실패하면 `null` 을 돌려준다.

```ts
import yaml from "js-yaml";

function _parseFrontmatterMeta(raw: string): Partial<DesignMeta> | null {
  const match = raw.match(/^---\n([\s\S]*?)\n---/);
  if (!match) return null;
  try {
    return yaml.load(match[1]) as Partial<DesignMeta>;
  } catch {
    return null;
  }
}
```

`null` 은 "이 경로로는 못 뽑았다"는 신호다. 예외를 던지지 않는다. 예외를 던지면 파이프라인 전체가 멈추는데, 지금 필요한 건 조용히 다음 경로로 넘어가는 것이었다.

## fallback 은 별도 함수로

frontmatter가 없는 문서에서 최소한의 메타를 건지려면 결국 본문을 봐야 했다. 정확도는 떨어져도 상관없다. 카탈로그 카드에 뭐라도 뜨는 게 아무것도 없는 것보단 낫다.

세 가지만 잡기로 했다. 제목은 첫 H1. 설명은 어느 정도 길이가 되는 첫 단락. 색상은 문서 어디에 있든 hex를 순서대로 긁는다.

```ts
function _parseMarkdownFallback(raw: string): Partial<DesignMeta> {
  const title = raw.match(/^#\s+(.+)/m)?.[1]?.trim() ?? "";

  const paragraphs = raw
    .split(/\n{2,}/)
    .map((p) => p.replace(/[#`*_]/g, "").trim())
    .filter((p) => p.length >= 50);
  const description = paragraphs[0] ?? "";

  const colorList = [...raw.matchAll(/#([0-9A-Fa-f]{6})\b/g)].map(
    (m) => `#${m[1].toUpperCase()}`
  );

  return { title, description, colorList };
}
```

길이 50자 기준은 임의로 잡았다. "사용법", "예시" 같은 소제목 아래 짧은 한 줄이 description으로 잡히는 걸 피하려고 넣은 하한이다. 이 숫자가 정확한 근거가 있는 건 아니라, 다른 카탈로그에 붙일 땐 다시 재봐야 한다.

hex 색상을 "본문 등장 순서"로 뽑는 게 예상 밖의 이득이었다. frontmatter가 있는 문서에도 `colors:` 키가 빠져 있는 경우가 있었는데, 이 방식은 예시 코드나 설명 문장 안에 적힌 색까지 자연스럽게 주워온다. 파서 종류에 상관없이 색은 잡힌다.

## 렌더러를 파서 구현에서 떼어냈다

처음 렌더러는 이런 식이었다.

```ts
const primary = meta.colors[0];
const secondary = meta.colors[1];
```

`colors` 키가 있을 거란 가정이 코드 곳곳에 박혀 있었다. fallback으로 들어온 문서는 그 키 자체가 없으니 swatch가 통째로 비었다.

파서 두 개를 만들면서 출력 스키마를 하나로 통일했다. `colors` 라는 키는 버리고, 어느 파서에서 왔든 `colorList` 라는 배열 하나만 채운다. 렌더러는 그것만 본다.

```ts
const swatches = meta.colorList?.slice(0, 5) ?? [];
```

`slice(0, 5)` 는 방어용이다. 어떤 문서엔 hex가 30개 넘게 박혀 있었는데, UI가 그걸 다 그리려다 카드 폭이 터지는 걸 봤다.

## 남은 것

파서를 통과한 뒤 71개 문서에서 title이 비는 케이스는 사라졌다. description은 여전히 애매하다. fallback 경로로 들어온 문서 중 몇 개는 첫 단락이 "이 문서는 ~에 대해 설명한다" 같은 자기지시적 문장이라, 카드 설명으로는 정보가 부족하다. 이건 아직 못 고쳤다.

hex 색상 순서가 "본문 등장 순서"인 것도 완전히 옳은 규칙은 아니다. 어떤 문서는 warning·error 색을 먼저 예시로 든 다음 primary를 나중에 설명한다. 그 경우 카드 첫 swatch가 빨간색이 된다. 색 역할을 캡션에서 추론하는 걸 다음에 볼 생각이다.