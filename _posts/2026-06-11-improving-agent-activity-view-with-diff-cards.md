---
title: "AI 에이전트 작업 로그를 그냥 로그가 아니라 IDE식 변경 카드로 보여주기"
date: 2026-06-11 17:00:00 +0900
categories: [Frontend]
tags: [diff-ui, ai-agent, activity-log]
---

# 도입 (왜 이 글)

AI 에이전트가 파일을 수정했다고 하면 나는 믿어야 할까? 로그 패널에 `EditFile: src/utils.py` 한 줄만 찍혀있으면 솔직히 반신반의한다. 실제로 뭘 어떻게 바꿨는지 모르면, 에이전트가 잘 한 건지 아닌지 판단할 수가 없다.

그래서 하네스(Harness — AI 에이전트 실행 환경) Activity 뷰를 손봤다. 목표는 단순했다: Codex나 GitHub Copilot Workspace처럼, 변경 사항을 **IDE식 diff 카드**로 보여주는 것. 이 글은 그 구현 과정에서 마주친 세 가지 문제와 해결책을 정리한 구현 가이드다.

## 문제 1 — 이벤트가 두 개인데 카드는 하나여야 한다

에이전트 SDK는 도구 호출 전후로 두 가지 이벤트를 발행한다.

- `PreToolUse` — 도구 실행 직전, 입력값 포함
- `PostToolUse` — 도구 실행 직후, 출력값 포함

로그를 그대로 렌더링하면 같은 작업이 두 줄로 나뉜다. 사용자 입장에서는 "이거 같은 거 아닌가?" 혼란이 온다.

해결책은 **fingerprint**로 두 이벤트를 묶는 것이다.

```python
# fingerprint = md5(tool + input_json)
import hashlib, json

def make_fingerprint(tool_name: str, input_data: dict) -> str:
    raw = tool_name + json.dumps(input_data, sort_keys=True)
    return hashlib.md5(raw.encode()).hexdigest()[:12]
```

`PreToolUse`가 들어오면 fingerprint를 키로 카드를 생성하고, 이후 같은 fingerprint의 `PostToolUse`가 오면 기존 카드를 업데이트한다. 카드 수 = 실제 작업 수. 깔끔하다.

diff 파일도 같은 키로 저장해두면 나중에 다시 펼쳐볼 수 있다.

```
.harness/diffs/<fingerprint>.diff
```

## 문제 2 — 기본 뷰는 요약만, 클릭하면 diff 전체

모든 diff를 펼쳐놓으면 화면이 폭발한다. 대형 리팩토링 한 번이면 수백 줄이다.

구조를 이렇게 잡았다.

| 상태 | 표시 내용 |
|------|----------|
| 기본(접힘) | 파일명, +N / -N 요약, 첫 3줄 미리보기 |
| 펼침 | syntax-highlighted diff 전체 |

펼쳐진 diff는 무한히 길어지지 않도록 스크롤 컨테이너로 감쌌다.

```css
.diff-body {
  max-height: 420px;
  overflow-y: auto;
  font-family: 'JetBrains Mono', monospace;
  font-size: 13px;
}
```

`420px`은 카드 3~4개가 동시에 화면에 들어오면서도 긴 diff를 읽을 수 있는 실용적인 타협점이었다.

## 문제 3 — Prism이 없을 때의 폴백

Syntax highlighting(문법 강조 — 코드의 키워드·문자열 등을 색으로 구분하는 기능)은 Prism.js를 쓰려 했다. 그런데 하네스는 CDN 없는 오프라인 환경에서도 동작해야 했다. Prism이 런타임에 로드 안 되면 그냥 흰 텍스트만 나온다.

그래서 경량 폴백을 직접 만들었다.

```javascript
function _simpleHighlight(code, lang) {
  if (typeof Prism !== 'undefined') {
    return Prism.highlight(code, Prism.languages[lang] || Prism.languages.plain, lang);
  }
  // 폴백: diff 전용 regex 하이라이터
  return code
    .split('\n')
    .map(line => {
      if (line.startsWith('+')) return `<span class="hl-add">${escHtml(line)}</span>`;
      if (line.startsWith('-')) return `<span class="hl-del">${escHtml(line)}</span>`;
      if (line.startsWith('@@')) return `<span class="hl-hunk">${escHtml(line)}</span>`;
      return escHtml(line);
    })
    .join('\n');
}
```

diff 형식은 `+` / `-` / `@@` 세 가지 패턴이 대부분이라 regex 10줄로 충분했다. Prism이 있으면 Prism을 쓰고, 없으면 폴백이 조용히 작동한다.

## 마치며

결과적으로 Activity 뷰는 이렇게 바뀌었다.

- **전**: 이벤트 타임라인 나열, 텍스트만
- **후**: 작업 단위 카드, 미리보기 + 클릭 → diff 펼침, 색상 강조

에이전트가 한 일을 한눈에 검토할 수 있게 되니 "이 변경 맞아?" 확인에 드는 시간이 확 줄었다. UI 한 번 잘 만들어두면 에이전트 신뢰도가 올라간다 — 이게 이번 작업의 진짜 교훈이다.
```