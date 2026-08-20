---
title: "외부망 커밋을 내부망 GitLab에 옮기는 git format-patch 실전 흐름"
date: 2026-08-20 17:00:00 +0900
categories: [Tooling]
tags: [git, format-patch, devops, network-isolation, tooling]
---

외부망에서 짠 코드를 내부망 GitLab에 올려야 하는 상황이 생겼다. 직접 push는 당연히 막혀 있다. USB든 파일 공유든 어떤 방식이든 파일을 넘길 수는 있다. 문제는 파일을 그냥 복사하면 git 히스토리가 날아간다는 거다. 커밋 메시지, 작성자, 날짜 — 이걸 보존하지 않으면 내부망 GitLab에서는 "누가 언제 왜 짰는지"를 전혀 알 수 없다.

`git format-patch`가 이걸 해결한다. 패치 파일(patch file) — 커밋 하나를 이메일 형식으로 직렬화한 텍스트 파일 — 을 만들고, 반대편에서 `git am`으로 재생하는 방식이다.

---

## 패치 만들기

외부망 머신에서, 옮기고 싶은 커밋 범위를 지정해 패치를 뽑는다.

```bash
# <base>는 포함하지 않고 <end>는 포함한다
git format-patch <base>..<end> --stdout > changes.patch
```

`--stdout`을 붙이면 커밋이 여러 개여도 파일 하나로 합쳐진다. 붙이지 않으면 `0001-...patch`, `0002-...patch` 식으로 커밋마다 파일이 생긴다. 파일 하나로 옮기는 편이 전달이 단순하다.

`<base>`에는 내부망 저장소와 공통으로 있는 커밋을 넣는다. 아직 공유되지 않은 커밋만 뽑아야 하기 때문이다. 모르겠으면 `git log --oneline`으로 확인하고 갈린 지점을 찾는다.

---

## 주의: working tree 상태로는 패치를 못 뽑는다

처음에 놓치기 쉬운 부분이다. `git format-patch`는 **커밋된 내용**을 대상으로 한다. 파일을 수정했더라도 `git commit`을 하지 않으면 패치에 포함되지 않는다.

옮기기 전에 외부망 쪽에서 모든 변경이 커밋되어 있는지 확인한다.

```bash
git status
# nothing to commit, working tree clean 이어야 한다
```

"아직 커밋 안 했는데 일단 넘겨볼게"는 통하지 않는다.

---

## 내부망에서 패치 적용하기

패치 파일을 내부망 머신으로 옮겼으면, 적용할 브랜치로 이동한 뒤 `git am`을 실행한다.

```bash
git am changes.patch
```

이게 전부다. 커밋 메시지, 작성자 이름과 이메일, 타임스탬프가 그대로 재현된다. `git log`로 확인하면 외부망에서 만든 커밋이 히스토리에 줄줄이 들어와 있다.

---

## 브랜치가 꼬여 있을 때

내부망 저장소의 로컬 상태가 외부망과 다른 경우가 있다. 예를 들어 내부망 GitLab에는 `develop` 브랜치가 있는데 로컬에는 그 tracking branch가 없는 경우다.

```bash
# 원격 브랜치를 로컬로 가져온다
git checkout -b develop origin/develop
```

패치를 `main`에 먼저 적용했다가 `develop`으로 옮겨야 할 때는 cherry-pick을 쓴다.

```bash
git cherry-pick main
```

이때 `main`은 브랜치 이름이 아니라 "main 브랜치의 HEAD 커밋"을 가리킨다. 커밋이 여러 개면 해시를 직접 지정하거나 범위로 넘긴다.

최종적으로 내부망 GitLab에 올리는 건 평범한 push다.

```bash
git push origin develop
```

---

## 실제로 쓰다 보면

패치 적용 중 충돌이 나면 `git am`이 멈춘다. 충돌을 해소하고 `git am --continue`로 재개하거나, 포기하려면 `git am --abort`다. 충돌 없이 깔끔하게 들어가려면 패치를 뽑기 전에 내부망 쪽 최신 상태를 외부망에 먼저 반영해두는 게 낫다 — 근데 네트워크가 분리된 환경에서 그게 쉽지 않다는 게 이 상황의 본질적인 불편함이다.

아직 해결하지 못한 게 하나 있다. 패치를 여러 번 나눠 옮기다 보면 base 커밋을 매번 확인해야 하는데, 양쪽 저장소의 공통 조상을 자동으로 특정하는 깔끔한 방법을 아직 찾지 못했다. `git merge-base`가 있긴 한데, 원격과 로컬이 분리된 상황에서 어떻게 맞춰 쓸지는 더 실험해봐야 한다.