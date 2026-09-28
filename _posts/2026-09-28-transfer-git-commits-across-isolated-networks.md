---
title: "push할 수 없는 환경에서 커밋 옮기기: 패치 전달과 브랜치 복구"
date: 2026-09-28 17:00:00 +0900
categories: [DevOps]
tags: [git, git-patch, network-isolation, git-am, devops]
---

망이 분리돼 있다. 인터넷도 없고, 원격 저장소도 없다. 그래도 커밋은 옮겨야 한다.

이런 상황이 처음이었다면 막막했을 텐데, 이미 한 번 겪었다. 그래도 이번엔 두 군데서 막혔다. 오류 메시지가 달랐고, 원인도 달랐다.

---

## 출발점: 패치 만들기

`git format-patch`는 원격이 필요 없다. 로컬 커밋만 있으면 된다.

```bash
git format-patch <base>..<end> --stdout > transfer.patch
```

base는 대상 환경에 이미 있는 커밋, end는 옮기고 싶은 마지막 커밋. `--stdout`으로 파일 하나로 모아 USB에 담았다. 이건 잘 됐다.

문제는 그 다음이었다.

---

## 첫 번째 막힘: `src refspec` 오류

패치를 받은 환경에서 `git am`을 실행했다.

```bash
git am transfer.patch
```

커밋이 찍혔다. 그런데 `git push origin develop`을 치니까 이런 메시지가 나왔다.

```
error: src refspec develop does not match any
```

처음엔 원격 저장소 문제인가 싶었다. 그런데 `git branch -a`를 쳐보니 로컬에 `develop`이 없었다. `remotes/origin/develop`은 있었다. 원격 추적 브랜치(remote-tracking branch)는 존재했는데, 그것을 로컬 브랜치로 착각하고 있었던 것이다.

`git am`은 현재 체크아웃된 브랜치에 커밋을 올린다. 나는 `main`에 체크아웃된 상태로 패치를 적용했다.

```bash
git status -sb
## main...origin/main
```

커밋은 `main`에 들어가 있었다. 해결 순서는 이랬다.

```bash
# 원격 추적 브랜치에서 로컬 브랜치를 만든다
git checkout -b develop origin/develop

# main에 잘못 들어간 커밋을 가져온다
git cherry-pick main

# 이제 push
git push origin develop
```

원격 추적 브랜치로부터 로컬 브랜치를 만드는 것과, 서버에 새 브랜치를 만드는 것은 다른 작업이다. `git checkout -b develop origin/develop`은 서버에 아무것도 하지 않는다. 로컬에서만 일어나는 일이다.

---

## 두 번째 막힘: pre-receive 500

push를 했더니 이번엔 다른 오류였다.

```
remote: Internal Server Error (500)
remote: error: hook declined to update refs/heads/develop
```

`src refspec` 오류와 다르다. 이건 로컬 문제가 아니라 서버 쪽 pre-receive 훅이 내부 API를 호출하다가 실패한 것이다. 저장소 서버가 커밋을 받으면서 외부 시스템을 찌르는 구조였고, 그 시스템이 500을 던졌다.

로컬에서 할 수 있는 게 없다. 서버 관리자에게 연락하거나, 훅이 호출하는 외부 시스템 상태를 확인해야 한다. 이 구분을 못 하면 로컬 브랜치를 이리저리 만들면서 시간을 보낸다.

---

## 정리하면

| 오류 | 원인 | 해결 위치 |
|------|------|-----------|
| `src refspec does not match any` | 로컬 브랜치 부재 | 로컬 |
| `hook declined / 500` | 서버 훅 내부 오류 | 서버 |

메시지만 읽어도 구분된다. `src refspec`은 git이 로컬에서 내는 말이고, `hook declined`는 서버가 내는 말이다.

---

패치 전달은 완료했다. 앱 수준 검증과 서버 500 이슈는 아직 남아 있다. 훅이 호출하는 내부 API가 정확히 어떤 조건에서 실패하는지 아직 못 봤다.