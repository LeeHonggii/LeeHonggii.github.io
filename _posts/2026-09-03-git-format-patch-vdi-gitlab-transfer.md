---
title: "외부망 커밋을 VDI GitLab에 옮기며 배운 git format-patch 실전 흐름"
date: 2026-09-03 17:00:00 +0900
categories: [DevOps]
tags: [git, format-patch, vdi, devops]
---

망이 분리돼 있다. 외부망 노트북에서 짠 코드를 VDI 안의 GitLab으로 옮겨야 한다. USB도 안 되고, 공유 드라이브도 없다. 패치 파일 하나만 허용된 경로를 통과할 수 있다.

처음엔 "그냥 코드를 복사해서 VDI에서 다시 커밋하면 되지 않나?" 싶었다. 실제로 한 번은 그렇게 했다. 커밋 메시지를 다시 타이핑하고, 작성자가 바뀌었다는 걸 나중에야 알았다. `git format-patch`를 쓰는 이유가 거기 있다.

---

## 패치로 커밋을 내보내기

`git format-patch`는 커밋을 `.patch` 파일로 직렬화한다. 커밋 메시지, 작성자, 타임스탬프까지 그대로 담긴다. 원격 저장소가 전혀 없어도 된다. 로컬 커밋만 있으면 충분하다.

외부망에서:

```bash
git format-patch <base-commit>..<end-commit> --stdout > transfer.patch
```

`<base-commit>` 은 옮기고 싶은 커밋 직전의 해시다. `--stdout` 으로 하나의 파일로 합쳐서 뽑으면 경로 이동이 편하다. 커밋이 여러 개면 각각 번호가 붙은 파일로 분리되는데, `--stdout`을 쓰면 그게 하나로 합쳐진다.

이 파일을 VDI 안으로 들여온다. 그다음:

```bash
git am transfer.patch
```

`git am`(apply mailbox)은 패치 안의 커밋 정보를 그대로 복원한다. 작성자도, 메시지도 바뀌지 않는다.

---

## 브랜치를 착각했다

여기서 실수를 했다. VDI 저장소를 클론하고 나서 바로 `git am`을 돌렸는데, 현재 브랜치가 `main`이었다.

이유가 있었다. `git branch`로 확인했을 때 로컬에 `develop`이 없었다. 원격에는 `origin/develop`이 있는데, 로컬 추적 브랜치를 따로 만들지 않았던 것이다. 클론 직후에는 `main`(또는 기본 브랜치)만 로컬에 생기고, 나머지는 원격 참조(`origin/develop`)만 있다.

```bash
# origin/develop을 추적하는 로컬 브랜치 만들기
git checkout -b develop origin/develop
```

이걸 빠뜨리고 `git am`을 실행하면 `main`에 커밋이 올라간다. 그게 내 상황이었다.

---

## cherry-pick 으로 수습

`main`에 붙어버린 커밋을 `develop`으로 옮겨야 했다.

```bash
git checkout develop
git cherry-pick <commit-hash>
```

`cherry-pick`은 지정한 커밋의 변경 내용을 현재 브랜치에 새 커밋으로 적용한다. 커밋 해시는 `git log main`으로 확인했다.

이후 `develop`에만 push했다. `main`의 잘못된 커밋은 그 환경에서는 강제 push가 막혀 있어서 되돌리지 못했다. 나중에 팀에 공유하고 처리했다.

---

## GitLab pre-receive 500

push 과정에서 GitLab이 500을 반환했다. 처음엔 패치가 잘못됐나 싶었다. `git am`은 에러 없이 끝났고, 로컬 커밋 로그도 정상이었다.

500은 GitLab 서버 쪽 문제였다. pre-receive hook이 실패하거나, 서버 리소스 이슈거나. 내 커밋 내용이나 패치 형식 문제가 아니었다.

인프라 팀에 확인 요청을 넣고 잠시 기다렸다. 잠시 후 같은 커밋이 아무 문제 없이 push됐다.

이게 중요한 분리다. 커밋/패치 문제인지 서버 문제인지. 로컬에서 `git log`가 정상이고 `git am`이 성공했다면, push 실패는 대부분 서버 쪽을 먼저 의심해야 한다.

---

지금도 이 흐름이 매번 깔끔하진 않다. 망 분리 환경이라 패치 파일을 어떤 경로로 옮기느냐가 그때마다 다르고, 가끔 커밋 범위를 잘못 지정해서 패치가 비어 있기도 하다. `--stdout` 없이 뽑으면 파일이 여러 개 생기는데, 순서를 틀리면 `git am`이 중간에 멈춘다.

정작 `format-patch` 자체보다 브랜치 상태를 눈으로 확인하는 습관이 더 중요했다. `git branch -a`로 로컬과 원격을 같이 보고, 지금 내가 어디 있는지 먼저 확인하고 `am`을 돌리는 것.
```