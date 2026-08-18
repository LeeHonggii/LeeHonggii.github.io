---
title: "Claude Code 자동 업데이트 실패, 알고 보니 npm과 Homebrew 권한 충돌이었다"
date: 2026-07-21 17:00:00 +0900
categories: [Tooling]
tags: [claude-code, npm, homebrew, macos, permissions]
---

터미널을 열었는데 상단에 회색 글씨 한 줄이 얌전히 앉아 있었다.

```
Auto-update failed. Run `claude update` to update manually.
```

무시하고 하던 일을 마저 했다. 며칠 뒤 다시 눈에 밟혔다. `claude update` 를 쳐봤다. 조용히 실패했다. `npm i -g @anthropic-ai/claude-code` 도 마찬가지. 로그를 뒤지니 `EACCES`. 퍼미션 거부.

## 어디에 깔려 있는가부터

`EACCES` 를 봤을 때 반사적으로 `sudo` 를 붙이고 싶어졌다. 참았다. 그 반사가 애초에 이 문제를 만든 원인일 가능성이 컸다. 먼저 어디에 깔려 있는지부터 봤다.

```bash
which claude
# /opt/homebrew/bin/claude

npm root -g
# /opt/homebrew/lib/node_modules
```

Homebrew 로 깐 Node 위에 npm 이 얹혀 있고, 그 npm 이 global 패키지를 Homebrew 트리 안쪽에 쌓는 구조. Homebrew 는 원칙적으로 `/opt/homebrew` 아래를 **현재 사용자 소유**로 유지한다. 그 원칙이 깨져 있으면 npm 이 rename 을 할 자리에서 튕긴다.

## 소유권을 봤다

```bash
ls -la /opt/homebrew/Cellar/node/
```

일부 하위 디렉터리가 `root:admin` 으로 잡혀 있었다. 나머지는 내 계정. 균일하지 않다는 점이 오히려 힌트였다. 언젠가 한 번, 검색해서 나온 명령을 그대로 붙여 넣은 `sudo npm install -g ...` 이 있었을 것이다. 그 한 번이 특정 경로의 소유권을 `root` 로 바꿔놓고, 이후 npm 이 그 경로에 손을 댈 때마다 조용히 실패해 온 것이다.

npm 이 global 패키지를 갱신할 때는 새 버전을 임시 경로에 풀고 최종 자리로 `rename` 한다. 그 최종 자리 소유자가 `root` 면, 내 셸에서 시작한 npm 프로세스는 그 자리에 못 들어간다. 그래서 자동 업데이트 훅이 조용히 실패하고, 회색 안내 한 줄만 남긴 것이다.

## 재설치가 아니라 chown

처음 든 유혹은 "Node 를 다 지우고 다시 깔자" 였다. 그 유혹을 접은 이유는 두 가지다. 하나, Node 위에 얹힌 global 패키지가 여러 개였고 그중 재현하기 귀찮은 게 있었다. 둘, 원인이 뚜렷했다. 소유권만 되돌리면 되는 문제에 재설치는 과했다.

`sudo` 를 이번 한 번만 쓴다. 되돌리는 용도로만.

```bash
sudo chown -R "$(whoami):admin" /opt/homebrew/Cellar/node
sudo chown -R "$(whoami):admin" /opt/homebrew/lib/node_modules
sudo chown -R "$(whoami):admin" /opt/homebrew/bin
```

`$(whoami)` 로 계정명을 박아 넣지 않았다. 스크립트로 남겨도 다른 머신에서 그대로 돌게 하고 싶었다. 그룹은 macOS 기본인 `admin`.

그 다음 재설치.

```bash
npm i -g @anthropic-ai/claude-code
```

이번엔 아무 말 없이 끝났다. 아무 말이 없다는 게 정답이다.

## claude doctor

Claude Code 에는 자체 진단 커맨드가 있다. 이번 일이 있고 나서야 존재를 알았다.

```bash
claude doctor
```

Node 버전, 설치 경로, 권한 상태를 한 번에 훑어준다. 소유권을 되돌리기 **전에** 먼저 돌려봤어야 했다. 다음번엔 그 순서로 갈 것이다.

## 남은 것

`sudo npm install` 은 실행한 그 시점의 한 줄로 끝나지 않는다. Homebrew 트리 안쪽 어딘가에 `root` 소유의 자국을 남기고, 그 자국이 몇 달 뒤 자동 업데이트가 실패하는 형태로 다시 튀어나온다. 인과 사이 거리가 멀어서 원인을 짐작하기 어렵다.

아직 안 한 것: 애초에 npm global 을 Homebrew 트리 밖으로 빼서 사용자 홈 아래에 두는 편이 더 깔끔했을 수 있다. `npm config set prefix ~/.npm-global` 쪽. 지금은 급하지 않아서 놔뒀는데, 다음에 Node 를 크게 올릴 일이 생기면 그때 같이 옮길 것 같다.