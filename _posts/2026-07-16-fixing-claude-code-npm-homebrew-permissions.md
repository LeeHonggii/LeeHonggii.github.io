---
title: "Claude Code 자동 업데이트 실패, 알고 보니 npm과 Homebrew 권한 충돌이었다"
date: 2026-07-16 17:00:00 +0900
categories: [DevOps]
tags: [claude-code, npm, homebrew, macos, permissions]
---

터미널을 열자마자 눈에 들어온 건 빨간색 한 줄이었다.

```
Auto-update failed: EACCES: permission denied, rename '/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/...'
```

처음엔 흘려봤다. Claude Code 쪽에서 뭐 하나 잘못 배포했겠거니. 다음 날 아침에도 똑같은 메시지가 떠 있길래 그제서야 자세를 고쳐 앉았다.

## 도구를 의심하기 전에

`EACCES`. 익숙한 문구다. 도구 버그가 아니라 파일 시스템이 "너는 이 파일 못 건드려"라고 말하는 것. 그런데 나는 매일 이 도구를 쓰고 있고, 처음 설치할 땐 아무 문제 없었다. 그 사이 뭔가 바뀐 것이다.

어느 `claude` 바이너리가 실행되는지부터 봤다.

```bash
$ which claude
/opt/homebrew/bin/claude

$ npm root -g
/opt/homebrew/lib/node_modules
```

둘 다 Homebrew 관할 경로. 여기까지는 자연스럽다. Node를 Homebrew로 깔았으니 npm의 전역 설치 경로도 Homebrew 아래에 있는 게 정상이다.

문제는 그 안에 있었다.

```bash
$ ls -la /opt/homebrew/lib/node_modules/@anthropic-ai/
drwxr-xr-x  root  admin  ...  claude-code
```

`root`. 소유자가 내 계정이 아니라 root였다. 이 순간 원인이 잡혔다.

## 언젠가의 sudo 한 방

Homebrew Node 환경에서는 npm 전역 디렉터리가 현재 유저 소유여야 한다. 그래야 `claude`가 자기 자신을 업데이트할 때 `sudo` 없이 파일을 갈아끼울 수 있다.

그런데 나는 언젠가 — 아마 다른 패키지를 깔다가 EACCES가 났을 때 — `sudo npm install -g ...`로 밀어붙인 적이 있다. 그 순간 npm이 만든 하위 디렉터리 몇 개가 root 소유로 바뀌었고, 그 뒤로 그 위에 설치된 모든 것이 같은 소유권을 물려받았다.

자동 업데이트는 현재 유저 권한으로 돈다. root 소유 파일 위에 rename을 시도하면 EACCES. 아침마다 실패했던 이유.

## 되돌리기

이건 그냥 소유권을 원위치시키면 끝나는 문제다.

```bash
sudo chown -R "$(whoami):admin" /opt/homebrew/lib/node_modules
sudo chown -R "$(whoami):admin" /opt/homebrew/Cellar/node/$(ls /opt/homebrew/Cellar/node/)
```

`-R` 붙여서 하위까지. `$(whoami)` 로 현재 유저 이름을 자동으로 꽂았다. group은 macOS 기본인 `admin`.

되돌린 뒤 재설치 한 번.

```bash
npm i -g @anthropic-ai/claude-code
```

그리고 `claude doctor`. Claude Code가 자체적으로 Node 버전·설정 경로·권한을 점검해주는 명령이다. 여기서 초록불이 뜨면 실제로 다음 자동 업데이트가 성공할 상태가 된다.

## 남는 규칙

정리해보면 Homebrew Node를 쓰는 환경에서 지켜야 할 건 하나뿐이다. **npm 전역 설치에는 sudo를 붙이지 않는다.** EACCES가 뜨면 sudo로 우회하지 말고 소유권부터 본다. `ls -la`로 root가 찍혀 있으면 chown으로 되돌린다.

`sudo npm install -g` 한 줄이 편해 보이지만, 그건 지금 이 문제를 미래의 나에게 미루는 것이다. 몇 달 뒤 아침에 붉은 EACCES를 보게 만드는.

아직 확인 못 한 게 하나 있다. npm이 아니라 pnpm이나 volta 같은 걸로 Node를 관리하면 이 문제 자체가 안 생기는 걸로 알고 있는데, 다음 맥 세팅 때 실제로 갈아타 볼지는 아직 못 정했다.