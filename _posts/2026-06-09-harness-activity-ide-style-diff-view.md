---
title: "AI 에이전트 로그를 카드 더미에서 IDE 스타일 diff 뷰로 바꾸기"
date: 2026-06-09 17:00:00 +0900
categories: [Frontend]
tags: [claude-code, activity-view, diff-ui, obsidian-plugin]
---

# 도입 (왜 이 글)

AI 코딩 에이전트를 쓰다 보면 어느 순간 이런 생각이 든다.  
"얘가 파일을 뭘 어떻게 바꿨지?"

기존 Activity view는 도구 호출 이벤트를 카드 한 장씩 쌓아 보여주는 구조였다. `PreToolUse` (도구 실행 직전) 이벤트가 카드 하나, `PostToolUse` (도구 실행 직후) 이벤트가 또 카드 하나. 같은 `Edit` 호출이 두 장으로 나뉘어 나타나니 스크롤만 길어졌다. 실제로 어떤 줄이 바뀌었는지 보려면 카드를 열어야 했고, 열어도 raw JSON이 기다리고 있었다.

이번에 이 뷰를 손봤다. 목표는 단순했다: **펼치지 않아도 뭘 바꿨는지 알 수 있게**.

---

## 문제 1: 같은 호출이 카드 두 장으로 나오는 중복

`PreToolUse` / `PostToolUse` 이벤트는 Claude Code Harness가 도구 실행 전·후로 각각 발행한다. 둘 다 같은 `tool_name`과 입력 파라미터를 가지므로, 피드에 그대로 렌더링하면 동일 작업이 두 번 나타난다.

해결은 **fingerprint 기반 dedup**이었다.

```js
// renderFeed() 내부 — Pre/Post 이벤트 묶기
function buildFingerprint(event) {
  const input = event.input ?? {};
  return `${event.tool_name}::${input.file_path ?? ""}::${input.old_string?.slice(0, 40) ?? ""}`;
}

const seen = new Map();
for (const ev of rawEvents) {
  const fp = buildFingerprint(ev);
  if (ev.type === "PreToolUse") {
    seen.set(fp, ev);
  } else if (ev.type === "PostToolUse" && seen.has(fp)) {
    // Post 이벤트로 Pre를 덮어쓰기 (결과 포함)
    seen.set(fp, { ...seen.get(fp), ...ev, _merged: true });
  } else {
    seen.set(fp + ev.id, ev); // 독립 이벤트
  }
}
const dedupedEvents = [...seen.values()];
```

fingerprint는 `tool_name + file_path + old_string 앞 40자` 조합이다. 완벽하진 않지만 실제 에이전트 루프에서 같은 파일을 수십 초 안에 두 번 다르게 편집하는 경우는 드물어 충분히 실용적이었다.

---

## 문제 2: 카드만 봐서는 무슨 변경인지 모름

카드 타이틀에 `Edit · src/utils.ts` 정도만 보여줘도 어떤 줄이 바뀌었는지는 알 수 없다. **_extractChangePreview()** 함수를 만들어 카드 본문 상단에 1~2줄 미리보기를 꽂았다.

```js
function _extractChangePreview(event) {
  const { tool_name, input } = event;

  if (tool_name === "Edit" || tool_name === "MultiEdit") {
    const edits = tool_name === "MultiEdit"
      ? (input.edits ?? [])
      : [{ old_string: input.old_string, new_string: input.new_string }];

    return edits.slice(0, 2).map(e => {
      const removed = (e.old_string ?? "").split("\n")[0];
      const added   = (e.new_string ?? "").split("\n")[0];
      return `- ${removed.trim()}\n+ ${added.trim()}`;
    }).join("\n");
  }

  if (tool_name === "Write") {
    const lines = (input.content ?? "").split("\n");
    return lines.slice(0, 3).map(l => `+ ${l}`).join("\n");
  }

  return null;
}
```

카드에 이 결과를 `<pre class="change-preview">` 블록으로 넣었다. 에이전트가 50개 파일을 고쳤어도 스크롤 한 번에 각 파일의 첫 번째 변경 라인이 눈에 들어온다.

---

## 문제 3: Obsidian 런타임엔 PrismJS가 없다

코드 블록에 syntax highlight(구문 강조)를 입히려면 보통 PrismJS나 highlight.js 같은 라이브러리를 쓴다. 그런데 Obsidian 플러그인 환경은 번들 크기와 런타임 격리 때문에 외부 라이브러리를 그냥 `import`할 수 없는 경우가 많다.

그래서 **정규식 기반 fallback** `_simpleHighlight()`를 직접 짰다.

```js
function _simpleHighlight(code, lang) {
  const escape = s => s.replace(/&/g,"&amp;").replace(/</g,"&lt;");
  let s = escape(code);

  const rules = {
    keyword : /\b(def|class|return|import|from|if|else|for|in|const|let|var|function|async|await|export|default)\b/g,
    string  : /(["'`])((?:\\.|(?!\1)[^\\])*)\1/g,
    comment : /(\/\/[^\n]*|#[^\n]*|\/\*[\s\S]*?\*\/)/g,
    number  : /\b(\d+\.?\d*)\b/g,
  };

  s = s.replace(rules.comment, '<span class="hl-comment">$1</span>');
  s = s.replace(rules.string,  '<span class="hl-string">$1$2$1</span>');
  s = s.replace(rules.keyword, '<span class="hl-keyword">$1</span>');
  s = s.replace(rules.number,  '<span class="hl-number">$1</span>');
  return s;
}
```

Python, JS/TS, CSS, JSON, Bash를 모두 같은 규칙셋으로 처리한다. 완벽한 파서는 아니지만 키워드·문자열·주석·숫자 정도만 색이 달라져도 가독성이 눈에 띄게 올라간다.

---

## 마치며

카드 더미 로그에서 IDE 스타일 diff 뷰로 바꾸면서 세 가지를 정리했다.

| 문제 | 해결 방법 |
|------|-----------|
| Pre/Post 이벤트 중복 | fingerprint dedup |
| 변경 내용 불투명 | `_extractChangePreview()` 인라인 미리보기 |
| 외부 라이브러리 없음 | regex fallback highlight |

AI 에이전트가 많은 파일을 고칠수록 "뭘 바꿨는지" 빠르게 파악하는 UI의 가치는 커진다. 완벽한 diff 도구를 새로 만들 필요는 없다. 기존 카드 뷰에 fingerprint dedup 하나, 미리보기 두 줄만 추가해도 리뷰 속도가 체감상 확 달라진다.
```