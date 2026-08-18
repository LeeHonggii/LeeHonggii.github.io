---
title: "Claude Code 자동 업데이트가 EACCES로 터질 때: npm과 Homebrew 경로 충돌 디버깅"
date: 2026-08-13 17:00:00 +0900
categories: [Tooling]
tags: [claude-code, npm, homebrew, macos, eacces]
---

터미널에서 Claude Code를 쓰다가 어느 날 배너에 붉은 줄이 떴다.

```
Auto-update failed: EACCES: permission denied, mkdir '/opt/homebrew/...'
```

처음 든 생각은 엉뚱했다. 백업 도구가 파일을 잠갔나. 보안 프로그램이 새 바이너리를 막나. 뭘 최근에 설치했더라. 삽질이 30분쯤 이어졌고, 로그를 다시 읽어보고 나서야 힌트가 이미 메시지 안에 있었다는 걸 알았다. `mkdir` 이 실패한 경로가 `/opt/homebrew/...` 였다는 것.

## 로그가 가리키는 곳부터 본다

`EACCES` — Error ACCESs. 유닉스에서 권한 없는 디렉터리에 쓰기를 시도했을 때 뜨는 흔한 코드다. 여기서 배운 것: `sudo npm install` 을 반사적으로 치기 전에, 에러가 지목한 경로의 **소유자**를 먼저 봐야 한다.

Claude Code는 `@anthropic-ai/claude-code` 라는 npm 패키지로 배포된다. 자동 업데이트가 돌면 npm global 경로에 새 파일을 내려받는데, 그 경로가 내 소유가 아니면 여기서 막힌다.

내 맥에서 상태를 정리해보면 대충 이랬다.

| 항목 | 값 |
|---|---|
| `which claude` | `/opt/homebrew/bin/claude` |
| `npm root -g` | `/opt/homebrew/lib/node_modules` |
| 문제 디렉터리 소유자 | `root:wheel` |
| 현재 사용자 | 내 계정 |

Homebrew로 Node를 깔았고, npm의 global prefix가 그 안쪽을 가리키고 있고, 그런데 그 디렉터리 주인이 root였다. 어느 시점엔가 `sudo brew ...` 로 뭔가를 설치했던 흔적. 정확히 언제였는지 기억은 안 난다.

## 확인만 하는 명령 세 줄

`sudo` 를 치기 전에 이것부터.

```bash
which claude
npm root -g
ls -la /opt/homebrew/lib/node_modules
ls -la /opt/homebrew/Cellar/node/
```

세 번째, 네 번째 줄이 핵심이다. 여기 나오는 소유자가 내가 아니라 `root` 면, 원인은 확실해진다. 반대로 소유자가 정상인데도 `EACCES` 가 나면 그건 다른 이야기(예: 파일시스템 락, SIP 관련 경로) — 그건 이 글의 범위는 아니다.

## 소유권을 되돌린다

해결 자체는 짧다. 꼬인 디렉터리의 주인을 다시 나로 돌리는 것.

```bash
sudo chown -R "$(whoami):admin" /opt/homebrew/Cellar/node/25.5.0
sudo chown -R "$(whoami):admin" /opt/homebrew/lib/node_modules
```

버전 번호(`25.5.0` 자리)는 각자 `ls /opt/homebrew/Cellar/node/` 로 확인해서 쓴다. 여러 버전이 남아 있으면 다 바꿔주는 편이 낫다. 어느 게 지금 링크된 건지 매번 헷갈리니까.

그 다음 재설치.

```bash
npm i -g @anthropic-ai/claude-code
```

이번엔 `sudo` 없이 붙는다. 붙어야 정상이다. 만약 여기서 또 `EACCES` 가 뜬다면 `chown` 범위가 부족한 것 — 다시 `ls -la` 로 어디 소유자가 아직 root인지 본다.

마지막으로 상태 점검.

```bash
claude doctor
```

설치 경로, 권한, API 연결 등을 한 번에 훑어주는 자기 진단 명령이다. 여기서 clean 하게 나오면 다음 자동 업데이트도 그냥 통과한다.

## sudo brew 를 안 쓰는 이유

Homebrew 공식 문서가 반복해서 강조하는 규칙 하나가 있다.

> Homebrew 명령에는 `sudo` 를 붙이지 않는다.

이유는 위에서 겪은 대로다. `sudo brew install` 한 번이 Cellar 안쪽 소유자를 root로 바꿔버리고, 그 뒤로 npm이 같은 트리를 건드릴 때마다 권한 충돌이 난다. 원인은 몇 달 전 한 줄이고, 증상은 오늘 뜬다. 이런 조합이 제일 진단하기 싫다.

## 좀 더 근본적으로: npm prefix 를 분리

Homebrew Node 트리 안에 npm global 을 얹는 구조 자체가 살짝 위험하다. 두 패키지 매니저가 같은 디렉터리를 공유하는 셈이니까. 굳이 얽힐 이유가 없다면 npm global prefix 를 홈 아래로 빼는 편이 깔끔하다.

```bash
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
# ~/.zshrc
export PATH="$HOME/.npm-global/bin:$PATH"
```

이렇게 하면 Homebrew가 관리하는 Node 바이너리와, npm이 관리하는 전역 패키지 트리가 물리적으로 다른 곳에 있게 된다. 소유권이 꼬여도 서로 영향을 안 준다.

다만 나는 아직 이 분리까지는 안 했다. 지금 설정으로도 자동 업데이트가 돌아가는 걸 확인했고, 굳이 새 PATH 를 zshrc 에 끼워두면 다른 도구들 사이드이펙트를 또 재봐야 해서. 다음 번에 같은 종류의 권한 사고가 한 번 더 뜨면, 그때 분리로 넘어갈 것 같다.