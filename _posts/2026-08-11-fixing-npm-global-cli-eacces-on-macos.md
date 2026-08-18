---
title: "Claude Code 자동 업데이트가 깨진 진짜 이유: npm이 아니라 소유권이었다"
date: 2026-08-11 17:00:00 +0900
categories: [DevOps]
tags: [claude-code, npm, homebrew, macOS, ownership]
---

터미널 하단에 며칠째 같은 배너가 붙어 있었다.

```
Update available! Run npm i -g @anthropic-ai/claude-code to update.
```

시키는 대로 쳤다. `EACCES: permission denied`. 앞에 `sudo`를 붙였다. 잘 깔린 것처럼 보였다. 다음 세션을 켜니 배너가 또 있었다. 버전은 그대로.

npm이 이상한가 싶어서 캐시를 지워봤다. `npm cache clean --force`. 아무것도 안 바뀌었다. 재설치, 재설치, 재설치. 세 번째 `sudo npm i -g`를 치던 순간에 이상하다는 감이 왔다. **매번 성공하는데 매번 실패하는** 상태는 npm의 문제가 아니다. npm은 시킨 대로 하고 있다. 다음 실행 주체가 그걸 만지지 못하는 게 문제다.

## 파일 시스템부터 봤다

에러 메시지를 더 읽는 대신 `ls -la` 를 쳤다.

```bash
$ which claude
/opt/homebrew/bin/claude

$ ls -la /opt/homebrew/bin/claude
lrwxr-xr-x  1 root  admin  ...  claude -> ../Cellar/node/25.5.0/bin/claude

$ ls -la /opt/homebrew/lib/node_modules/@anthropic-ai/
drwxr-xr-x  root  admin  claude-code
```

`root`. 심링크도, 실제 패키지 디렉터리도 전부 `root` 소유였다. 내 계정으로 실행되는 Claude Code 프로세스가 자동 업데이트를 시도하면, 이 파일들을 덮어쓰지 못한다. 배너가 안 사라지는 이유가 여기 있었다. 자동 업데이트가 조용히 실패하고, 다음 실행 때 다시 "업데이트 있음" 을 안내하는 무한 루프.

## sudo 한 번의 대가

거슬러 올라가 보면 원인은 몇 주 전이다. 처음 설치할 때 그냥 `npm i -g` 를 쳤고, `EACCES` 가 떠서 반사적으로 `sudo` 를 붙였다. 그 순간 `/opt/homebrew/lib/node_modules` 아래의 파일 소유자가 내 계정에서 `root` 로 바뀌었다.

Homebrew는 자기가 관리하는 디렉터리가 **로그인 사용자 소유**라고 가정한다. `brew` 자체가 sudo 없이 도는 이유가 그거다. 그런데 그 안에서 `sudo npm` 이 한 번 돌면 그 가정이 깨진다. 이후로는:

- 사용자 권한으로 돌리는 `npm i -g` → `root` 소유 파일을 못 건드림 → `EACCES`
- 그럼 또 `sudo` 를 붙이게 됨 → `root` 소유 파일이 더 늘어남

한 번의 `sudo` 가 다음 `sudo` 를 강제하는 구조다. 여기서 빠져나오려면 소유권을 되돌리는 수밖에 없다.

## 복구

Node 셀러 경로 전체와 전역 패키지 디렉터리, 그리고 심링크가 사는 `bin` 까지 사용자로 되돌렸다.

```bash
brew list --versions node
# node 25.5.0

sudo chown -R "$(whoami):admin" /opt/homebrew/Cellar/node/25.5.0
sudo chown -R "$(whoami):admin" /opt/homebrew/lib/node_modules
sudo chown -R "$(whoami):admin" /opt/homebrew/bin
```

그 다음 **sudo 없이** 재설치.

```bash
npm i -g @anthropic-ai/claude-code
claude doctor
```

`claude doctor` — Claude Code 내장 진단 명령어(설치 경로·권한·버전을 한 번에 점검) — 가 초록 표시로 도배됐다. 배너도 사라졌다. 다음날 자동 업데이트가 조용히 도는 걸 로그에서 확인했다.

## 남은 애매함

한 가지 아직 안 재본 게 있다. `nvm` 으로 Node를 관리하는 환경이면 사용자 홈 아래에 설치되니 애초에 이 문제가 안 생긴다. 그럼 팀 세팅을 nvm 기반으로 통일하는 게 나은지, 아니면 Homebrew Node를 유지하되 "sudo npm 금지" 를 문서로 박아두는 게 나은지 — 아직 결론을 안 냈다. 다음에 새 맥을 세팅할 때 nvm 쪽으로 한 번 밀어보고 다시 판단하려 한다.

에러 메시지가 `EACCES` 라고 외쳐도, 그게 곧 "권한을 올려라" 는 뜻은 아니다. 오히려 `sudo` 를 붙이는 순간 문제가 시작되는 경우가 있다. `ls -la` 5초가 `sudo` 30분보다 먼저다.