---
title: "AI 코딩 로그를 그냥 이벤트 목록이 아니라 IDE식 변경 카드로 바꾸기"
date: 2026-06-17 17:00:00 +0900
categories: [Frontend]
tags: [diff-ui, ai-agent, obsidian-plugin]
---

# 도입 (왜 이 글)

AI 에이전트가 파일을 수정할 때, 내 화면엔 이런 목록이 흘러내렸다.

```
[PreToolUse] Edit
[PostToolUse] Edit
[PreToolUse] Edit
[PostToolUse] Edit
```

같은 도구 호출이 두 줄씩 찍히고, 어떤 파일의 몇 번째 줄이 바뀌었는지는 카드를 직접 열어봐야 알 수 있었다. 에이전트가 10개 파일을 손댔다면, 나는 20개 카드를 하나하나 클릭해야 했다. 이건 로그가 아니라 소음이다.

이 글은 그 소음을 **IDE 스타일 diff 카드**로 바꾼 과정을 정리한다. Obsidian 플러그인 런타임처럼 외부 라이브러리를 마음대로 쓸 수 없는 환경까지 포함해서.

---

## 중복 카드 문제: PreToolUse + PostToolUse를 fingerprint로 합치기

> **fingerprint** — 이벤트를 고유하게 식별하는 해시 값. 같은 도구 호출은 같은 fingerprint를 갖게 만들어 중복을 제거한다.

에이전트 하네스(harness)는 도구 호출 전후로 각각 이벤트를 던진다. 화면에 그대로 렌더링하면 `PreToolUse` + `PostToolUse` 쌍이 카드 두 장으로 보인다.

해결책은 단순하다. 이벤트가 들어올 때마다 **도구 이름 + 입력 JSON**으로 fingerprint를 만들고, 이미 같은 fingerprint의 카드가 있으면 새 카드를 추가하는 대신 기존 카드를 업데이트한다.

```js
// fingerprint 생성 (md5 대신 간단한 해시도 무방)
const fingerprint = md5(event.tool + JSON.stringify(event.input));

if (cardMap.has(fingerprint)) {
  cardMap.get(fingerprint).update(event); // PostToolUse 데이터로 갱신
} else {
  const card = new ActivityCard(event, fingerprint);
  cardMap.set(fingerprint, card);
  container.appendChild(card.el);
}
```

이것만으로 카드 수가 절반으로 줄었다. `PreToolUse` 단계에서 "진행 중" 배지를 보여주고, `PostToolUse`가 오면 "완료"로 바꾸면 상태 피드백도 자연스럽게 따라온다.

---

## 변경 미리보기: 카드를 열지 않아도 흐름이 보이게

카드 수를 줄였어도, 각 카드가 "Edit · src/index.ts" 한 줄만 보여준다면 클릭 없이는 맥락을 알 수 없다.

`_extractChangePreview(input)` 함수는 `Edit`, `MultiEdit`, `Write` 이벤트의 input에서 **실제로 바뀐 라인의 요약**을 뽑아낸다.

```js
function _extractChangePreview(input) {
  if (input.old_string && input.new_string) {
    // Edit: 삭제된 첫 줄 / 추가된 첫 줄
    const removed = input.old_string.split('\n')[0];
    const added   = input.new_string.split('\n')[0];
    return { removed, added };
  }
  if (input.content) {
    // Write: 새 파일 첫 3줄
    return { added: input.content.split('\n').slice(0, 3).join('\n') };
  }
  return null;
}
```

카드 헤더 아래에 이 미리보기를 한두 줄 노출해두면, 에이전트가 무슨 작업을 했는지 스크롤만으로 파악된다.

---

## diff 렌더링: Prism 없는 환경의 fallback

> **Prism** — 웹 환경에서 코드 신택스 하이라이팅(syntax highlighting)에 자주 쓰이는 JavaScript 라이브러리.

Obsidian 플러그인은 `window.Prism`이 없을 수 있다. CDN을 직접 로드하면 보안 정책에 걸리기도 한다. 그래서 `_renderEditDiff`는 Prism 존재 여부를 런타임에 확인하고, 없으면 간단한 정규식 하이라이터로 폴백(fallback)한다.

```js
function _renderEditDiff(event, filePath) {
  const { removed, added } = _extractChangePreview(event.input);
  const lang = filePath.split('.').pop(); // 'ts', 'py', ...

  const highlight = window.Prism
    ? (code) => Prism.highlight(code, Prism.languages[lang] ?? Prism.languages.plain, lang)
    : (code) => simpleRegexHighlight(code, lang);

  return `
    <pre class="harness-activity-diff-pre">
      <div class="diff-line removed">- ${highlight(removed)}</div>
      <div class="diff-line added">+ ${highlight(added)}</div>
    </pre>`;
}
```

스크롤 없이 전체가 보이도록 최대 높이를 CSS로 고정한다.

```css
.harness-activity-diff-pre {
  max-height: 420px;
  overflow-y: auto;
}
```

420px는 약 14줄 분량이다. 변경이 적은 경우엔 스크롤바가 아예 안 생기고, 긴 diff는 내부에서 스크롤된다.

---

## 마치며

로그 UI 개선은 기능 추가가 아니라 **인지 부하 감소**다. 카드가 반으로 줄고, 펼치지 않아도 변경 요약이 보이면 에이전트 작업을 리뷰하는 속도가 체감상 크게 달라진다.

세 가지를 요약하면:

| 문제 | 해결 |
|------|------|
| Pre/Post 중복 카드 | fingerprint로 같은 호출 합치기 |
| 카드 열어야 내용 파악 | `_extractChangePreview`로 헤더에 미리보기 |
| Prism 없는 런타임 | `window.Prism \|\| simpleRegexHighlight` 분기 |

AI 에이전트를 감독(supervise)하는 UI를 만들고 있다면, 이벤트를 1:1로 렌더링하는 것부터 의심해보자. 이벤트 스트림과 사람이 보는 카드는 다른 추상 수준이다.
```