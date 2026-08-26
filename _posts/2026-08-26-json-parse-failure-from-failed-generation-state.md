---
title: "JSON 파싱 에러인 줄 알았는데, 진짜 문제는 실패 상태를 저장하는 방식이었다"
date: 2026-08-26 17:00:00 +0900
categories: [Backend]
tags: [debugging, llm-pipeline, mysql, json]
---

생성 상태를 조회하는 API가 500을 냈다.

스택트레이스를 보니 `json.loads()` 에서 죽고 있었다. 처음에는 MongoDB 쪽 문제라고 생각했다. LLM 결과물이 BSON 직렬화 과정에서 깨지는 건 전에도 본 적 있었다. 그래서 MongoDB 연결 설정과 컬렉션 스키마부터 뒤지기 시작했다.

틀렸다.

코드 경로를 따라가 보니 오류가 나는 지점은 MongoDB가 아니었다. DB 접근 계층을 한 단계씩 올라가다 보면 실제로 조회하는 건 MySQL 테이블이었다. `json.loads` 가 실패하는 값은 그쪽에서 오고 있었다.

---

실제 데이터를 조회했다.

```sql
SELECT content_type, status, LEFT(content, 80)
FROM board_contents
WHERE board_id = '<id>';
```

결과를 보고 바로 원인을 알 수 있었다. `status` 가 실패 상태인 행들의 `content` 컬럼에 JSON이 아니라 평문 문자열이 들어 있었다.

```
"이미지 생성 실패: OpenAI API 오류 — insufficient_quota"
```

이런 식으로. 생성에 실패하면 어딘가에서 `content` 필드에 오류 메시지를 그대로 써넣고 있었다. 그리고 조회 API는 `content` 를 무조건 JSON으로 파싱하려 했다. 성공한 행은 실제로 JSON이 들어 있으니 잘 동작한다. 실패한 행을 조회하는 순간 파싱이 터진다.

버그가 두 군데에 있는 게 아니었다. 저장하는 쪽에서 타입 계약을 깬 게 조회하는 쪽에서 뒤늦게 폭발한 것이다.

---

저장하는 쪽 코드를 찾아보면 아마 이런 모양일 것이다.

```python
content = f"{content_type} 생성 실패: {error_message}"
```

이게 문제였다. 실패했을 때 `content` 에 평문을 넣어버렸다. `content` 는 정상 경로에서는 JSON 구조체인데, 예외 처리에서 그 계약을 무시했다.

고치는 방법은 단순하다. 실패 원인은 별도 필드(`error_message`, `failure_reason` 등)에 저장하고, `content` 는 실패 시 `null` 이나 빈 JSON 객체 `{}` 로 유지하거나, 아니면 조회 API가 `status` 를 먼저 확인해서 실패 행은 `json.loads` 를 건너뛰어야 한다.

---

그런데 이 조사를 하면서 다른 문제가 하나 더 보였다.

오류 메시지 안에 `insufficient_quota` 가 있었다. OpenAI API 크레딧이 소진된 상태였다. 그리고 API 키가 코드에 하드코딩되어 있었다. 환경변수로 바꾸는 작업을 develop 브랜치와 main 브랜치에 각각 적용했다.

크레딧 문제와 파싱 오류는 별개의 버그다. 크레딧이 있어도 파싱 오류는 발생했을 것이다. 실패한 생성이 쌓이면 쌓일수록 `content` 에 평문 메시지가 늘어났을 테니까.

---

갤러리 생성 실패 케이스는 또 달랐다.

이미지 검색 실패인 줄 알았는데 실제로는 LLM이 빈 plan을 생성했는데 파이프라인이 검증 플래그를 소비하지 않고 그냥 다음 단계로 넘어간 문제였다. 증상은 같아 보여도 원인이 전혀 다른 경로였다.

이 부분은 아직 고치지 않았다. 빈 plan을 감지해서 재시도할지, 아니면 hard fail시킬지 결정이 안 됐다. 재시도는 비용이 들고, hard fail은 UX가 나빠진다. 어느 쪽이 맞는지는 실제 빈 plan 발생 빈도를 재봐야 알 수 있다. 아직 그 수치가 없다.