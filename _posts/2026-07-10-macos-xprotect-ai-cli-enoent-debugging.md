---
title: "어제까지 되던 AI CLI가 ENOENT로 죽었다면 macOS가 지운 걸 수도 있다"
date: 2026-07-10 17:00:00 +0900
categories: [Tooling]
tags: [macos, gatekeeper, xprotect, codex, automation]
---

산출물이 0개로 끝난 아침에 알아챘다.

자동화 파이프라인은 매일 새벽에 도는 것들이 몇 개 있는데, 그중 하나가 며칠째 빈 결과를 뱉고 있었다. cron은 exit 0. 스크립트 로그도 끝까지 흘렀다. 그런데 결과 폴더가 비어 있다. 이런 실패가 제일 사납다. 실패했다는 신호조차 없다.

## exit 0인데 결과가 없다

처음엔 스크립트가 이상한 조건 분기를 탔나 싶어서 로그를 위에서부터 다시 읽었다. 별 게 없었다. 그래서 스크립트를 손으로 실행해 보니 그제서야 진짜 에러가 나왔다.

```
spawn /Users/me/node_modules/@openai/codex-darwin-arm64/vendor/.../codex ENOENT
```

`ENOENT`. errno로는 "No such file or directory". Node의 `child_process.spawn`이 자식 프로세스를 띄우려는데 실행 파일이 그 자리에 없다는 뜻이다. 파이썬에서 잡힐 땐 subprocess 계열이 조용히 stderr만 뱉고 상위에선 exit 0으로 끝나기도 한다. 그게 "결과 0개, 로그는 정상"의 정체였다.

## npm 패키지는 멀쩡한데 바이너리만 없다

이상한 건 여기부터였다. `npm list`는 codex를 정상으로 표시했고 `node_modules/@openai/codex-darwin-arm64` 폴더도 있었다. 그런데 폴더 안을 뒤져 보니 정작 실행돼야 할 플랫폼 바이너리 하나만 사라져 있었다.

```bash
codex --version
# zsh: command not found: codex
```

PATH에 걸어둔 심링크도 깨진 상태. 재설치를 했다. 잘 되는가 싶었는데 하루 뒤에 같은 자리에서 또 사라졌다. 이때 npm 얘기가 아니라는 걸 받아들이기 시작했다.

## macOS가 지운다는 가능성

macOS에는 서명 없는 바이너리를 다루는 두 레이어가 있다. Gatekeeper는 첫 실행 때 서명·공증(notarization)을 확인해서 막는 쪽이고, XProtect는 알려진 악성 패턴을 백그라운드에서 훑어 격리하거나 삭제하는 쪽이다.

문제는 XProtect가 조용히, 사용자 몰래 돈다는 것. 그리고 XProtect의 정의(definition)는 백그라운드 업데이트로 계속 바뀐다. 어제까지 무사했던 바이너리가 오늘 새 정의에 걸려서 없어질 수 있다. npm이 다시 받아 놓으면, 다음 스캔 때 또 지운다. 원인을 모른 채 `npm install`만 반복하면 이 루프에서 못 나온다.

이 소재는 npm 패키지 관리자 관점에선 이해가 잘 안 되는 실패다. "패키지는 있는데 안에 있어야 할 파일이 없다"는 상태는 npm이 만들지 않는다. OS가 만든 상태다.

## quarantine 떼고 스스로 서명하기

우선 확장 속성(extended attribute)에 quarantine이 붙어 있으면 뗀다. 인터넷을 통해 내려온 파일에 `com.apple.quarantine`이 붙는데, Gatekeeper가 이걸 보고 첫 실행 때 검사한다.

```bash
xattr -l <codex-binary-경로>
# com.apple.quarantine: 0081;...;...

xattr -dr com.apple.quarantine <codex-binary-경로>
```

그다음이 ad-hoc signing. Apple 개발자 계정이 없어도 자기 자신으로 임시 서명을 할 수 있다. 외부 배포용이 아니라 로컬 실행용이면 이걸로 충분히 통과한다.

```bash
codesign --force --deep --sign - <codex-binary-경로>
```

`--sign -`의 `-`가 self-signed를 의미한다. `--deep`은 안에 포함된 헬퍼들까지 훑어서 서명한다. 그리고 실행 확인.

```bash
codex --version
```

버전 문자열이 나오면 살아난 거다. 이 세 줄이 실제 복구의 전부였다. 진짜 짜증나는 부분은 복구가 아니라 원인을 의심하기까지 걸린 시간이었다.

## 자동화 안에서 이걸 어떻게 감지할까

한 번 겪고 나서 파이프라인에 얇은 방어선을 넣었다. 실행 직전에 바이너리 존재와 실행 가능 여부를 먼저 확인하고, 없으면 큰 소리로 실패하게 만들었다. exit 0으로 조용히 끝나는 게 제일 나빴으니까.

체크리스트는 이렇게 갖고 다닌다.

- `codex --version`이 정상 종료하는지
- `xattr -l`에 quarantine이 다시 붙었는지
- `/var/log/system.log` 또는 Console.app에서 XProtect / `XProtectRemediator` 로그가 최근에 돌았는지
- 증상이 시작된 날이 macOS 마이너 업데이트나 XProtect 정의 업데이트 직후인지

마지막 항목이 실제로 결정적이었다. XProtect 정의 업데이트 타임스탬프와 파이프라인이 빈 결과를 뱉기 시작한 날이 같은 날이었다.

## 아직 안 해본 것

서명 없는 바이너리를 XProtect가 계속 지우는 근본 원인을 우회할 방법은 두 가지 정도 남겨뒀다. 하나는 CLI를 npm 패키지가 아니라 Homebrew 등 자체 배포 채널을 쓰는 걸로 바꾸는 것 — 그쪽은 공증된 바이너리를 쓰는 경우가 많다. 다른 하나는 파이프라인이 도는 사용자를 아예 XProtect 예외 대상으로 관리하는 방향인데, 이건 개인 장비 밖으로 나가는 순간 감당이 안 될 것 같아 아직 손대지 않았다.

당분간은 실행 전 존재 확인 + 실패 시 ad-hoc 재서명 스크립트로 버티는 중이다. 다음에 같은 증상을 다시 만나면 이 글부터 열어볼 것 같다.