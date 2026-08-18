---
title: "Claude Code 자동 업데이트 실패, 알고 보니 npm과 Homebrew 권한 충돌이었다"
date: 2026-07-10 17:00:00 +0900
categories: [DevOps]
tags: [claude-code, npm, homebrew, macos, permissions]
---

터미널을 열자마자 노란 배너가 떠 있었다.

```
Auto-update failed. Run `npm i -g @anthropic-ai/claude-code` to update manually.
```

그래, 수동으로 하지 뭐. 시키는 대로 쳤다.

```bash
npm i -g @anthropic-ai/claude-code
```

돌아온 건 `EACCES: permission denied`. 이때부터 이상하다는 감각이 들었다. 나는 Homebrew로 Node를 관리한다. Homebrew 방식은 애초에 `sudo` 없이 글로벌 설치가 되게끔 소유권을 사용자에게 두는 구조인데, 왜 권한 거부가 나지?

## 일단 어디에 뭐가 있는지부터

원인을 짐작하기 전에 위치를 찍었다.

```bash
which claude
# /opt/homebrew/bin/claude

npm root -g
# /opt/homebrew/lib/node_modules
```

경로 자체는 정상이었다. Apple Silicon Homebrew의 표준 위치. 그러면 권한이다.

```bash
ls -la /opt/homebrew/lib/node_modules/ | grep claude
# drwxr-xr-x  root  wheel  @anthropic-ai
```

`root:wheel`. 여기서 멈췄다. Homebrew 아래에 root 소유 디렉터리가 있다는 건, 언젠가 내가 `sudo`를 썼다는 뜻이다.

## 몇 달 전의 sudo 한 번

기억을 더듬으면 짚이는 게 있다. 예전에 어떤 npm 패키지가 안 깔린다고 짜증나서, 별생각 없이 `sudo npm install -g ...`를 한 번 친 적이 있다. 그 순간 해당 패키지의 파일들이 root 소유로 박혔고, 그 뒤로 그 트리 근처에서 rename·교체가 일어날 때마다 조용히 실패해 왔던 거다.

Claude Code의 자동 업데이터도 내부적으로 npm 패키지를 갈아치우면서 파일을 rename한다. 대상 디렉터리가 root 소유면 사용자 권한으로 도는 프로세스는 당연히 못 건드린다. 배너에 뜬 "Auto-update failed"는 그 실패의 마지막 얼굴이었을 뿐이다.

`sudo npm i -g`로 덮어씌우면 이번 순간은 성공한다. 하지만 그러면 root 소유 파일이 더 늘어난다. 미래의 자동 업데이트가 또 같은 이유로 죽는다. 그래서 해결은 반대 방향이어야 했다. **원래대로 사용자 소유로 되돌리는 것.**

## 소유권 복구

```bash
sudo chown -R "$(whoami):admin" /opt/homebrew/Cellar/node/
sudo chown -R "$(whoami):admin" /opt/homebrew/lib/node_modules/
sudo chown -R "$(whoami):admin" /opt/homebrew/bin/
```

세 군데를 다 훑은 이유는 각각 다르다. `Cellar/node`는 Homebrew가 실제 Node 바이너리와 라이브러리를 보관하는 실체, `lib/node_modules`는 npm 글로벌 패키지가 풀리는 자리, `bin`은 그 패키지들이 노출하는 실행 파일 심볼릭 링크(symlink)가 걸리는 곳이다. 셋 중 하나라도 root로 남아 있으면 업데이트 어딘가에서 다시 걸린다. 특히 `bin`을 빼먹기 쉬운데, 패키지 자체는 사용자 소유여도 `bin/claude` 링크가 root 소유면 링크 교체 단계에서 또 실패한다.

`$(whoami):admin`은 macOS에서 표준 관리자 그룹 조합이라 그대로 붙여넣어도 안전하다.

그 다음 재설치.

```bash
npm i -g @anthropic-ai/claude-code
```

이번엔 조용히 끝났다. `EACCES` 없이.

## claude doctor 로 마감

설치가 됐다는 것과, 잘 됐다는 것은 다르다. 마지막에 헬스체크를 걸었다.

```bash
claude doctor
```

`claude doctor`는 Claude Code CLI가 자체 진단으로 제공하는 명령이다. 실행 경로, 인증, API 도달 여부 같은 걸 한 번에 훑어준다. 전부 통과. 여기서 처음으로 "다 됐다"고 판단했다.

## 정리 대신, 앞으로의 규칙

이번 일에서 얻은 건 해결법 자체보다 규칙 하나였다.

Homebrew로 Node를 관리하는 환경에서는 `sudo npm install -g`를 쓰지 않는다. 권한 오류가 뜨면 반사적으로 `sudo`를 붙이는 대신, 먼저 `ls -la $(npm root -g)`로 소유자를 본다. root가 보이면 그게 원인이다.

찜찜하게 남은 건 하나 있다. 이번에 chown으로 되돌린 트리 바깥에도 예전 sudo 흔적이 더 있을지 모른다. 다른 글로벌 CLI가 조용히 자동 업데이트에 실패한 채로 몇 주 째 그 자리에 있을 수도 있다. 다음에 비슷한 배너를 또 만나면, 그때는 놀라지 않고 같은 순서로 소유자부터 보게 될 것 같다.