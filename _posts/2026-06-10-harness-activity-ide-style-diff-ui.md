---
title: "AI 에이전트 작업 로그를 IDE처럼 보기: Pre/Post 중복 제거와 코드 diff 미리보기"
date: 2026-06-10 17:00:00 +0900
categories: [Frontend]
tags: [ai-agent, diff-viewer, obsidian-plugin]
---

# 도입 (왜 이 글)

AI 에이전트가 파일을 수십 개 수정하고 나면, Activity 뷰에는 카드가 폭발한다. 문제는 같은 작업이 두 번씩 보인다는 것이다. `PreToolUse`(도구 실행 전)와 `PostToolUse`(도구 실행 후) 이벤트가 각각 카드로 렌더링되니, 사용자 입장에선 "Edit이 왜 두 개야?"가 된다. 거기에 실제로 어떤 줄이 바뀌었는지는 카드만 봐선 전혀 모른다. 이 글은 그 두 가지를 고친 과정이다.

## Pre/Post 이벤트를 한 카드로 합치기

`PreToolUse`와 `PostToolUse`는 동일한 도구 호출의 전·후 스냅샷이다. 이 둘을 구분하는 열쇠가 **fingerprint** — `session_id + fingerprint` 조합으로 만드는 고유 식별자다.

구현 방향은 단순하다. 카드를 렌더링하기 전에 이벤트 목록을 한 번 순회하면서, 같은 fingerprint를 가진 Pre/Post 쌍을 하나의 논리 단위로 묶는다.

```js
// Pre 이벤트를 Map에 먼저 등록
const seen = new Map(); // fingerprint → index

for (const evt of events) {
  const key = evt.session_id + evt.fingerprint;
  if (evt.type === 'PreToolUse') {
    seen.set(key, evt);
  } else if (evt.type === 'PostToolUse' && seen.has(key)) {
    evt._pre = seen.get(key); // Post 카드에 Pre 정보를 병합
    seen.delete(key);
  }
}
```

이렇게 하면 Post 카드 하나에 Pre 정보가 합쳐지고, 짝을 찾은 Pre 이벤트는 목록에서 제거된다. 결과적으로 같은 작업이 두 번 노출되던 문제가 사라진다.

## Edit 이벤트에서 변경 줄 미리 뽑기

카드에 "Edit: src/utils.ts" 만 보여주면 사용자는 파일을 직접 열어봐야 한다. `_extractChangePreview()` 함수를 만들어 `Edit`, `MultiEdit`, `Write` 입력 페이로드에서 핵심 변경 줄을 즉시 추출했다.

```js
_extractChangePreview(input) {
  if (input.new_string) {
    // Edit: 추가된 줄만 최대 5줄
    return input.new_string.split('\n').slice(0, 5).join('\n');
  }
  if (input.content) {
    // Write: 파일 앞부분 미리보기
    return input.content.split('\n').slice(0, 5).join('\n');
  }
  return null;
}
```

그리고 실제 diff가 저장된 `.harness/diffs/<fingerprint>.diff` 파일을 읽어 카드 하단에 인라인으로 펼치도록 `_renderEditDiff(evt, inp.file_path)` 를 연결했다. 클릭 한 번으로 어떤 줄이 추가·삭제됐는지 볼 수 있게 됐다.

## Prism 없는 환경을 위한 폴백 하이라이터

diff를 예쁘게 보여주려면 **syntax highlighting** (구문 강조)이 필요하다. 보통 Prism.js 같은 라이브러리를 쓰지만, 특정 플러그인 런타임에선 외부 라이브러리 로드가 제한된다.

그래서 `_simpleHighlight(lang, code)` 라는 폴백 함수를 만들었다. 정규식 기반이라 무겁지 않다.

```js
_simpleHighlight(lang, code) {
  if (!lang || !['js','ts','python','diff'].includes(lang)) return code;
  return code
    .replace(/&/g, '&amp;').replace(/</g, '&lt;') // XSS 방어
    .replace(/(\/\/.*)/g, '<span class="cm-comment">$1</span>')
    .replace(/(".*?"|'.*?')/g, '<span class="cm-string">$1</span>')
    .replace(/\b(const|let|var|def|return|import)\b/g,
             '<span class="cm-keyword">$1</span>');
}
```

diff 언어일 때는 `+`로 시작하는 줄은 초록, `-`로 시작하는 줄은 빨강으로 칠한다. Prism이 있으면 그걸 쓰고, 없으면 이 함수가 조용히 대신한다.

긴 diff는 기본적으로 접혀 있고 "▶ 12줄 더 보기" 같은 토글로 펼치도록 했다. 로그가 많을 때 카드 높이가 폭발하지 않도록.

## 마치며

에이전트 로그 UI를 개선하면서 배운 건, **이벤트 모델을 그대로 렌더링하면 안 된다**는 것이다. Pre/Post를 fingerprint로 합치고, 변경 줄을 카드에 바로 노출하고, 환경 제약을 고려한 폴백을 넣으면 — 사용자는 에이전트가 실제로 무엇을 했는지 파일을 열지 않고도 파악할 수 있다. 작은 UI 결정이 에이전트 신뢰도에 직접 영향을 준다.
```