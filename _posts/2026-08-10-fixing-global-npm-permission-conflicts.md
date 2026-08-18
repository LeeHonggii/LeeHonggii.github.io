---
title: "Claude Code 자동 업데이트가 깨졌을 때: npm, Homebrew, root 권한 충돌 추적기"
date: 2026-08-10 17:00:00 +0900
categories: [DevOps]
tags: [claude-code, npm, homebrew, permissions, macos]
---

터미널을 열면 Claude Code가 새 버전이 있다고 알린다. 자동 업데이트가 돈다. 근데 다음 세션에서도 똑같은 알림이 뜬다. 업데이트가 안 먹은 것이다.

수동으로 밀어보기로 했다.

```bash
npm i -g @anthropic-ai/claude-code
```

돌아온 건 이거였다.

```
npm error code EACCES
npm error syscall mkdir
npm error path /opt/homebrew/lib/node_modules/@anthropic-ai/claude-code
npm error errno -13
```

`EACCES`. 패키지 문제가 아니라 파일시스템이 나를 막아선 거다. 여기서 반사적으로 `sudo`를 붙이고 싶어지는데, 그게 이 사달의 원인이었다는 걸 나중에 깨달았다.

## 지금 어떤 claude가 도는지부터

문제 진단은 항상 "지금 무엇이 실제로 실행되는가"에서 시작한다. 경로가 어긋나 있으면 어떤 처방도 엉뚱한 곳을 고친다.

```bash
which claude
# /opt/homebrew/bin/claude

npm root -g
# /opt/homebrew/lib/node_modules
```

Homebrew로 Node를 깔았으니 global npm 경로는 `/opt/homebrew/lib/node_modules` 아래다. 이 경로의 소유자를 본 순간 그림이 그려졌다.

```bash
ls -l /opt/homebrew/lib/node_modules | grep anthropic
# drwxr-xr-x  root  admin  ...  @anthropic-ai
```

`root admin`. 내 사용자가 아니다. 언젠가 내가 `sudo npm install -g`를 쳤던 흔적이 여기 남아 있었다. 자동 업데이트는 `sudo`를 쓰지 않으니 조용히 실패할 수밖에.

## sudo npm은 왜 미래의 자신을 괴롭히는가

Homebrew의 전제는 "root 없이 돈다"이다. `/opt/homebrew` 아래 모든 것은 로그인 사용자 소유여야 한다. `sudo npm install -g`를 한 번 치면 그 패키지 트리 일부가 root 소유로 박히고, 이후 Homebrew가 Node를 업그레이드하거나 npm이 symlink를 다시 그릴 때마다 삐걱거린다.

더 나쁜 건, 이 오염이 **패키지 단위로 흩어진다**는 거다. `@anthropic-ai/claude-code`만 root일 수도 있고, 어떤 하위 디렉터리만 그럴 수도 있다. 나중에 다른 도구가 안 될 때 원인을 찾기 어렵다.

## 좁혀가면서 고친다

전체를 뒤엎기보다, 소유자를 확인하면서 필요한 만큼만 되돌리는 게 안전하다.

Node 본체부터. 버전은 `brew info node`로 뽑는다.

```bash
sudo chown -R "$(whoami):admin" /opt/homebrew/Cellar/node/25.5.0
```

그다음 global 모듈과 실행 파일 심볼릭 링크가 사는 곳.

```bash
sudo chown -R "$(whoami):admin" /opt/homebrew/lib/node_modules
sudo chown -R "$(whoami):admin" /opt/homebrew/bin
```

`$(whoami)`는 그대로 둔다. 사용자 이름을 하드코딩하지 않아도 된다.

이제 `sudo` 없이 다시.

```bash
npm i -g @anthropic-ai/claude-code
claude doctor
```

`claude doctor`는 Claude Code 내장 진단이다. 어떤 Node를 보고 있는지, auto-update 경로가 쓰기 가능한지 등을 요약해준다. 여기서 초록 불이 뜨면 자동 업데이트 루프가 다시 돌기 시작한다.

## 다음에 같은 함정을 피하는 법

복구를 해도 `sudo npm install -g`를 한 번 더 치면 그 순간 원점이다. 규칙은 단순하게 정리했다.

- Homebrew Node를 쓰는 한, global 설치에 `sudo`를 붙이지 않는다.
- `EACCES`가 뜨면 먼저 `ls -l`로 소유자를 본다. 처방은 그다음이다.
- 시스템 여러 Node 버전을 오갈 일이 있으면 `nvm`이나 `fnm`으로 사용자 홈에 두는 게 낫다. Homebrew와 경로가 겹치지 않으니 이 종류의 충돌 자체가 안 생긴다.

---

세 번째 항목 — Homebrew를 걷어내고 `fnm`으로 옮기는 건 아직 안 했다. 지금 로컬에 얹혀 있는 global 패키지 목록을 옮길 만한 값어치가 있는지 아직 판단이 서지 않아서. `chown`으로 응급처치한 상태가 얼마나 오래 버티는지 좀 지켜볼 생각이다. 다음 Node 메이저 업그레이드 때 또 뭐가 삐걱거리면 그때가 결정 시점이겠지.