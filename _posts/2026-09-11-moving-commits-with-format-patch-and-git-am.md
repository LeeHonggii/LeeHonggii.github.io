---
title: "직접 push가 막힌 환경에서 git format-patch로 커밋 옮기기"
date: 2026-09-11 17:00:00 +0900
categories: [DevOps]
tags: [git, format-patch, git-am, cherry-pick]
---

내부망 저장소에 직접 push 권한이 없었다. 외부에서 작업한 커밋을 넘겨야 했는데, 브랜치를 zip으로 압축해 메일로 보내는 건 선택지가 아니었다. 그렇다고 변경 파일만 복사하면 커밋 히스토리, 작성자 정보, 커밋 메시지가 다 날아간다.

`git format-patch`가 있었다.

---

## 패치 파일로 커밋을 묶는다

`git format-patch`는 커밋을 `.patch` 파일로 직렬화한다. 파일 하나에 diff뿐만 아니라 작성자, 날짜, 커밋 메시지까지 담긴다. 메일 포맷 기반이라 텍스트 파일이고, 옮기는 방법을 가리지 않는다.

```bash
git format-patch <base>..<head> --stdout > export.patch
```

`<base>`는 옮기고 싶은 커밋 직전 커밋이다. `<head>`는 마지막 커밋. `--stdout`으로 하나의 파일에 몰아넣었다. 커밋이 여러 개면 순서대로 다 들어간다.

파일을 내부망으로 옮긴다. 방법은 상황마다 다르다. 이때는 내부 파일 공유 도구를 썼다.

---

## 받는 쪽에서 `git am`으로 복원한다

`git am`(apply mailbox)은 `.patch` 파일을 읽어 커밋을 그대로 재현한다. 작성자도, 메시지도, 타임스탬프도 원본 그대로다.

```bash
git am export.patch
```

충돌이 없으면 그냥 된다. 충돌이 나면 `git am --show-current-patch`로 현재 위치를 확인하고, 수동으로 resolve한 뒤 `git am --continue`로 이어간다.

---

## 잘못된 브랜치에 적용됐다

문제는 내가 `main`에 체크아웃된 채로 `git am`을 돌린 거였다. 패치는 `develop`에 들어가야 했다.

커밋이 `main`에 찍혀버렸다. 당황했지만 복구는 어렵지 않았다.

```bash
git checkout -b develop origin/develop
git cherry-pick main
```

`cherry-pick`은 특정 커밋(여기선 방금 `main`에 잘못 올라간 커밋)을 현재 브랜치에 다시 적용한다. 작성자 정보는 유지된다. 커밋 해시만 바뀐다.

그 다음 `main`은 직전 상태로 되돌렸다.

```bash
git push origin develop
```

---

## pre-receive hook 500이 나왔다

push가 튕겼다. 오류 메시지가 `remote: error: 500`이었다. 처음엔 커밋 내용 문제인 줄 알았다. 패치 파일이 깨졌나, 커밋 메시지에 금지 문자가 있나 한참 의심했다.

아니었다. 서버 측 pre-receive hook이 내부 오류를 냈던 것이었다. 커밋 자체는 정상이었고, 잠시 후 재시도하니 그냥 됐다. 인프라 팀에 확인하니 훅 스크립트가 외부 서비스를 호출하는 구조였고, 그 서비스가 잠깐 응답을 못 했던 거였다.

500이 나오면 내 커밋을 의심하기 전에 재시도를 먼저 해보는 게 낫다.

---

아직 해결 못 한 부분이 있다. 커밋이 많을 때 `--stdout`으로 하나로 묶지 않고 번호를 붙인 여러 파일로 쪼개면(`git format-patch -N`) 충돌 지점을 커밋 단위로 특정하기 쉬워진다고 하는데, 실제로 충돌이 많은 상황에서 써본 적이 없다. 다음에 그런 상황이 생기면 비교해봐야겠다.