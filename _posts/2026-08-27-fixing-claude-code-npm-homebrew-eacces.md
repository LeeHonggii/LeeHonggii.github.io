---
title: "Claude Code 자동 업데이트가 EACCES로 막힐 때: npm과 Homebrew 권한 충돌 추적기"
date: 2026-08-27 17:00:00 +0900
categories: [Tooling]
tags: [claude-code, npm, homebrew, macos, permissions]
---

Claude Code가 세션 시작 때마다 "Auto-update failed"를 뱉기 시작했다.
처음엔 그냥 넘겼다. 어차피 동작은 했으니까.

이틀쯤 지나서 에러를 제대로 읽었다.

```
EACCES: permission denied, open '/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/...'
```

`/opt/homebrew` 경로. npm global이 Homebrew가 관리하는 Node를 쓰고 있다는 뜻이다.

---

## 언제부터 꼬인 건가

Homebrew로 Node를 설치할 때 대부분의 경로는 `$(whoami)`가 소유하도록 세팅된다. 그런데 언젠가 `sudo brew install`이나 다른 도구 설치 과정에서 Cellar 안의 Node 경로 일부가 `root` 소유로 바뀐 것 같았다. 내 경우엔 직접적인 원인을 특정할 수 없었다 — 그냥 어느 순간 그렇게 되어 있었다.

`ls -la /opt/homebrew/Cellar/node/` 로 확인해보니 일부 디렉터리의 owner가 `root`였다.

npm global install은 Node Cellar 경로 안에 패키지를 풀고, `bin/` 아래에 심볼릭 링크를 건다. owner가 `root`면 auto-update 때 쓰기 권한이 없다. EACCES는 여기서 나온다.

---

## 고쳐본 것들

처음엔 `node_modules` 쪽만 chown 했다.

```bash
sudo chown -R "$(whoami):admin" /opt/homebrew/lib/node_modules
```

업데이트는 한 번 됐다가 다음 세션에서 또 실패했다. `lib/node_modules`만 고쳐봐야 Cellar 안의 원본 경로가 여전히 root 소유면 결국 같은 에러가 난다.

Node 버전이 25.5.0이었으니 Cellar 경로를 통째로 잡았다.

```bash
sudo chown -R "$(whoami):admin" /opt/homebrew/Cellar/node/25.5.0
```

그 다음 재설치.

```bash
npm i -g @anthropic-ai/claude-code
```

마지막으로 확인.

```bash
claude doctor
```

이번엔 깔끔하게 통과했고, 그 이후로 auto-update 실패가 없다.

---

## 삽질 하나 더

이 과정에서 터미널에서 `!명령` 형태로 이전 명령을 재실행하려다가 명령이 엉뚱하게 치환된 적이 있다. zsh의 history expansion 때문이다 — `!`가 붙으면 이전 히스토리에서 매칭되는 걸 끌어온다. Claude Code 세션 안에서 `!` prefix로 shell 명령을 실행하는 습관이 있다면 터미널에서도 무심코 그렇게 입력하기 쉽다. 같은 실수를 할 것 같으면 그냥 history expansion을 꺼두는 게 낫다 (`setopt NO_BANG_HIST`).

---

아직 확인 못 한 게 있다. Homebrew Node를 완전히 버리고 nvm이나 fnm으로 Node 관리를 일원화하면 이 문제가 재발하지 않을지다. 지금 구성에서 Node를 바꾸면 다른 글로벌 도구들이 같이 영향받는 게 귀찮아서 미루고 있다. 언젠간 정리할 것 같기는 한데, 아직 안 했다.
```