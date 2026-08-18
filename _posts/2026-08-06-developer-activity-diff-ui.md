---
title: "AI 코딩 로그를 그냥 이벤트 목록이 아니라 코드 리뷰 카드처럼 보여주기"
date: 2026-08-06 17:00:00 +0900
categories: [Tooling]
tags: [devtools, logging, diff, frontend]
---

하루치 로그를 열었는데 스크롤이 끝나지 않았다.

에이전트가 파일 하나 고칠 때마다 `PreToolUse`(도구 호출 직전)와 `PostToolUse`(도구 호출 직후) 두 개가 붙어서 나온다. 파일 세 개를 만졌으면 여섯 장. 실제로는 같은 편집인데 화면에서는 계속 다른 카드다. 이걸 "읽는다"고 표현하기 어려웠다. 그냥 지나가는 걸 버티는 쪽에 가까웠다.

내가 원하던 건 PR 리뷰 화면 같은 것이었다. 어떤 파일이, 어떤 줄이, 왜 바뀌었는지. 카드 하나에 그게 다 들어와야 한다. 그래서 로그 뷰어를 뜯어 고쳤다. 세 군데를 손댔는데, 세 개가 각각 다른 문제를 풀었다.

## Pre/Post를 한 장으로

같은 도구 호출의 앞뒤 스냅샷은 결국 한 사건이다. 나누어 보여줄 이유가 없다. 문제는 "같은 호출"을 뭐로 정의하느냐였는데, 도구 이름과 파일 경로만으로는 부족했다. 짧은 시간 안에 같은 파일을 두 번 편집하는 경우도 있으니까. 그래서 타임스탬프를 2초 버킷으로 잘라서 붙였다.

```js
function makeFingerprint(event) {
  const tool = event.toolName ?? '';
  const path = event.input?.file_path ?? event.input?.path ?? '';
  const bucket = Math.floor(event.timestamp / 2000);
  return `${tool}::${path}::${bucket}`;
}

const seen = new Map();
for (const ev of rawEvents) {
  const fp = makeFingerprint(ev);
  if (!seen.has(fp)) {
    seen.set(fp, ev);
  } else {
    seen.get(fp).post = ev;
  }
}
```

2초라는 값은 임의로 골랐다. 더 짧으면 Post가 늦게 들어올 때 놓치고, 더 길면 연속 편집이 뭉친다. 로그 몇 개 훑어보고 정한 값이라 나중에 다시 재봐야 한다. 지금은 그대로 두고 있다.

카드가 반으로 줄어든 것보다, Pre의 "무엇을 하려 했는지"와 Post의 "무엇이 됐는지"를 같이 볼 수 있게 된 게 더 좋았다. 정보 밀도가 오히려 올라갔다.

## 미리보기 — diff 4줄

카드 안에서 diff 전체를 펼치니 이번엔 반대로 너무 길었다. 접어두면 아무것도 안 보이고, 펼치면 화면을 잡아먹는다. 중간이 필요했다.

`+`와 `-`로 시작하는 줄만 뽑고, `---`/`+++` 헤더는 버리고, 공백·괄호만 있는 줄도 버렸다. 남은 것 중 위에서 네 줄.

```js
_extractChangePreview(diffText) {
  if (!diffText) return null;
  const lines = diffText.split('\n');
  const significant = lines.filter(l => {
    if (!/^[+-]/.test(l)) return false;
    if (/^[+-]{3}/.test(l)) return false;
    const content = l.slice(1).trim();
    return content.length > 2 && !/^[\{\}\(\)\[\]]*$/.test(content);
  });
  return significant.slice(0, 4).join('\n') || null;
}
```

이 네 줄로 "함수 시그니처가 바뀌었네" 정도는 카드 접힌 채로 읽힌다. 4라는 숫자도 감으로 정한 거고, 5나 6이 나은지는 아직 안 재봤다. 카드 UI에서 두 줄짜리 접힘이 시각적으로 안정된다는 감이 있어서 4로 두었다.

한계는 있다. 삭제와 추가가 뒤섞이면 순서가 원래 diff와 달라 보여서 오해가 생긴다. 이건 아직 못 고쳤다.

## Prism을 안 붙이기로 한 이유

diff 색을 입히려면 신택스 하이라이터가 필요했다. Prism이 표준이고 편하지만 번들이 무거웠다. 그리고 로그 UI는 다른 곳으로 임베드될 가능성이 있어서 CDN 의존성이 껄끄러웠다. 대안이 있을까 잠깐 고민하다가, "로그에서 필요한 하이라이팅이 얼마나 되지?" 라고 물어보니 답이 나왔다. 문자열, 키워드, 숫자, 주석. 넷.

regex 네 개로 처리했다.

```js
_simpleHighlight(code, lang = '') {
  let html = code
    .replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');

  html = html.replace(/(["'`])(?:\\.|(?!\1)[^\\])*\1/g,
    '<span class="hl-string">$&</span>');
  html = html.replace(
    /\b(const|let|var|function|return|if|else|for|class|import|export|from|async|await)\b/g,
    '<span class="hl-keyword">$&</span>');
  html = html.replace(/\b(\d+)\b/g, '<span class="hl-number">$&</span>');
  html = html.replace(/(\/\/[^\n]*)/g, '<span class="hl-comment">$1</span>');

  return html;
}
```

```css
.diff-add  { background: #1a3a1a; color: #98c379; }
.diff-del  { background: #3a1a1a; color: #e06c75; }
.hl-keyword { color: #c678dd; }
.hl-string  { color: #98c379; }
```

이 regex는 정직하게 말하면 틀린다. 문자열 안에 `function`이 들어있으면 그것도 키워드로 칠하고, Python 주석(`#`)은 잡지 못한다. 그런데 로그 카드는 코드를 읽는 곳이 아니라 "무엇이 바뀌었나"를 확인하는 곳이라, 이 정도 오탐이 실사용에서 걸리지 않았다. 만약 걸리기 시작하면 그때 언어별 분기를 붙일 생각이다. 아직은 안 붙였다.

키워드 목록에 JS 계열만 넣은 것도 마찬가지다. Python 파일이 카드에 오면 색이 덜 들어가지만 못 읽을 정도는 아니었다.

---

셋을 다 붙이고 나니 하루치 작업이 카드 예닐곱 장 정도로 줄었다. 정확한 감축율은 재보지 않았다 — 원래 카드가 몇 장이었는지 로그를 남겨두지 않아서, 다음에 계측을 붙일 때 같이 볼 생각이다.

아직 애매하게 남은 게 있다. 하나의 도구 호출이 여러 파일을 만지는 경우(MultiEdit 같은) fingerprint가 파일 단위라 카드가 다시 쪼개진다. 이걸 도구 호출 단위로 묶을지, 지금처럼 파일 단위로 둘지 정하지 못했다. 파일 단위가 리뷰에는 더 자연스러운데, 시간순 흐름이 깨진다.

다음에 이 UI를 다시 열 때는 이걸 먼저 볼 것 같다.