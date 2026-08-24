---
title: "OpenAI 키는 살아있는데 호출은 실패한다: 크레딧 소진과 하드코딩 키 제거하기"
date: 2026-08-24 17:00:00 +0900
categories: [DevOps]
tags: [openai, api-key, devops, debugging]
---

콘텐츠 생성 파이프라인에서 500이 떨어지기 시작했다. 로그를 따라 올라가면 OpenAI 호출에서 죽고 있었다.

키 자체가 문제라고는 생각 못 했다. 발급한 지 얼마 안 됐고, 대시보드에서도 "활성" 상태였다. 그래서 처음엔 코드 쪽을 뒤집었다. 프롬프트 형식, 모델명 오타, 타임아웃 설정. 다 멀쩡했다.

---

## GET /v1/models가 200이어도 추론은 실패할 수 있다

OpenAI API 키 유효성 검사로 흔히 쓰는 게 모델 목록 조회다.

```bash
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer <api-key>"
```

200이 돌아왔다. 모델 목록도 멀쩡히 나왔다. 그래서 키는 살아있다고 판단했다.

틀렸다. 모델 목록 조회는 키 인증(Authentication)만 확인한다. 크레딧 잔액은 확인하지 않는다. 실제 추론 호출을 해봐야 크레딧 소진 여부가 드러난다. 에러 코드는 `insufficient_quota`, HTTP 상태는 429다. 이름이 "Too Many Requests"라서 처음엔 rate limit 문제인 줄 알고 재시도 로직을 의심했다.

진단을 제대로 하려면 최소 추론 호출을 직접 쏴봐야 한다.

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4o-mini", "messages": [{"role": "user", "content": "hi"}], "max_tokens": 1}'
```

여기서 `insufficient_quota`가 나오면 키가 아니라 크레딧이 문제다. 키를 교체해도 소용없고, 결제 수단과 크레딧을 봐야 한다.

---

## 하드코딩 키 19곳

원인을 확인하고 나서 키를 바꾸려다가 다른 문제가 보였다. 코드베이스 전체를 grep해보니 API 키가 문자열로 박혀있는 곳이 19곳이었다. 파일마다 같은 키가 반복되어 있었다.

이 구조의 문제는 단순하다. 키 하나 바꾸려면 19군데를 찾아서 고쳐야 한다. 그리고 그 키가 git 이력에 남는다.

바꾸는 방향은 환경변수로 일원화하는 것이었다. 코드에서는 `os.getenv`로 읽고, 실제 값은 배포 환경의 configmap이나 secret에 넣는다.

```python
from openai import AsyncOpenAI
import os

client = AsyncOpenAI(api_key=os.getenv('openai_key'))
```

이미 이 프로젝트의 다른 설정들은 이 방식을 쓰고 있었다. `ENABLE_DEBUG_PAYLOAD`, `ENABLE_TEST_PAGE`, `UPLOADS_DIR` 같은 것들은 전부 `os.getenv`로 읽으면서 OpenAI 키만 코드 안에 박혀있었다. 일관성이 없었던 것이다.

배포 환경에서는 키 값이 어디 있는지도 확인해야 했다.

```bash
kubectl -n <namespace> get configmap <configmap-name> -o yaml | grep -i openai
```

변수명이 코드의 `getenv` 인자와 정확히 일치해야 한다. 코드에서 `openai_key`를 읽는데 configmap에 `OPENAI_KEY`로 들어있으면 읽지 못한다. 대소문자까지 맞춰야 한다.

---

## develop과 main이 많이 갈라졌을 때

이 수정을 develop과 main 양쪽에 반영해야 했는데, 두 브랜치가 상당히 갈라진 상태였다. 파일 단위로 보면 겹치지 않는 변경이 많았다.

이런 경우 merge를 시도하면 충돌 해소 과정에서 의도치 않은 변경이 섞일 수 있다. 단순한 한 줄짜리 수정이 merge 커밋에 묻혀버리면 나중에 추적도 힘들어진다.

그래서 develop에서 수정 커밋을 하나 만들고, main에는 cherry-pick으로 올렸다.

```bash
# develop에서 특정 디렉토리의 변경만 스테이징
git add -u questboard/

# main으로 이동 후
git cherry-pick cef1001
```

수정 범위가 좁고 명확할 때 cherry-pick이 merge보다 안전한 이유는, 해당 커밋이 하는 일만 정확히 가져오기 때문이다. 나머지 브랜치 차이는 건드리지 않는다.

---

갤러리 생성 실패 건은 별도 원인이었다. 이미지 검색이 안 된다고 생각했는데, 실제로는 검색 호출 자체가 일어나지 않고 있었다. plan이 비어있고, 검증 플래그가 사용되지 않는 상태여서 검색 단계까지 도달을 못 하고 있었다. 이쪽은 아직 완전히 들여다보지 못했다.