---
title: "FastAPI에서 NestJS식 PaginationDto를 맞추려면 무엇이 바뀌나"
date: 2026-09-04 17:00:00 +0900
categories: [Backend]
tags: [FastAPI, Pydantic, Python, Pagination, REST-API]
---

프론트엔드 팀이 기대하는 응답 구조와 내가 짜놓은 구조가 다른 걸 발견하는 순간은 항상 배포 직전이다. 이번에도 그랬다.

---

## 무엇이 안 맞았나

기존 템플릿 목록 API는 이렇게 내보내고 있었다.

```json
{
  "status": "success",
  "count": 5,
  "templates": [...]
}
```

프론트 계약은 이랬다.

```json
{
  "totalCount": 276,
  "list": [...]
}
```

차이가 세 개다.

- `status` 필드 — 없애야 한다
- `count` → `totalCount` — 이름만 바뀐 게 아니다. 의미가 바뀐다
- `templates` → `list` — 키 이름

`status` 는 뺐다. 성공·실패를 `"success"` / `"error"` 문자열로 판별하던 로직을 HTTP 상태코드로 옮기는 게 맞다. FastAPI에서 예외 핸들러를 적절히 달아두면 (`@app.exception_handler(Exception)` 등) 500/503/422 가 정확하게 나온다. 굳이 응답 바디에 중복으로 실어줄 이유가 없다.

`count` 의 의미 변화가 더 중요했다. 원래 `count` 는 **이번 페이지에 실제로 내려온 아이템 개수**였다. `totalCount` 는 **전체 결과의 총 개수**다. SQL로 치면 `LIMIT` 적용 전에 `COUNT(*)` 를 따로 돌려야 한다는 뜻이다. 필드명 하나 바꾸는 게 아니라 쿼리가 하나 더 생긴다.

---

## offset 해석 차이

파라미터 이름은 둘 다 `offset` 과 `limit` 이었다. 그런데 NestJS 쪽 DTO는 이렇게 선언돼 있었다.

```typescript
offset: number = 1;
limit: number = 30;
```

기본값이 `0` 이 아니라 `1` 이다. 프론트는 offset 을 **1-based 페이지 번호**로 쓰고 있었다. 내 Python 코드는 그걸 그대로 SQL `OFFSET` 에 꽂고 있었고.

offset=1, limit=5 를 넣으면 기대값은 "1페이지, 5개" 인데, SQL에 `OFFSET 1` 을 주면 두 번째 행부터 5개가 나온다. 첫 번째 행이 날아간다.

변환식은 단순하다.

```python
skip = (offset - 1) * limit
```

offset=1 → skip=0, offset=2 → skip=5, offset=3 → skip=10. 맞다.

단순해서 오히려 놓치기 쉬운 종류의 버그다. 쿼리 결과가 `limit` 개 나오긴 하니까 얼핏 보면 정상처럼 보인다. 첫 페이지에서 첫 행이 빠졌다는 걸 알아채려면 데이터를 알아야 한다.

---

## Pydantic 쿼리 파라미터 선언의 함정

여기서 하나 더 걸렸다. FastAPI 에서 쿼리 파라미터를 Pydantic 모델로 묶어 선언할 때 함정이 있다.

`BaseModel` 을 그대로 라우터 인자로 받으면 FastAPI 는 그걸 request body 로 해석한다. GET 요청에 body 를 기대하는 셈이니, 클라이언트가 쿼리 스트링으로 보내면 파라미터가 전부 `None` 으로 들어온다. 에러도 안 난다. 기본값이 있으면 조용히 기본값으로 처리된다.

`Depends()` 로 감싸야 쿼리 스트링으로 바인딩된다.

실제 코드는 올려두지 않는다 — 관련 부분을 정확히 확인해서 다음 글에서 따로 다룰 생각이다. 지금은 "이 함정이 있다" 는 것만.

---

## 결과 응답 형태

최종적으로 내보내는 구조는 이렇다.

```json
{
  "totalCount": 276,
  "list": [
    { "id": "...", "title": "...", ... },
    ...
  ]
}
```

페이지네이션 파라미터 기본값은 offset=1, limit=30. 프론트 기본값과 맞췄다.

---

랜덤 추천에 페이지네이션을 붙이는 문제는 아직 열려 있다. seed 를 고정하면 페이지를 넘겨도 중복 없이 일관된 순서가 나오지만, 새로고침할 때마다 다른 순서를 원하는 요구사항과 충돌한다. seed 없이 순수 랜덤이면 페이지 경계에서 같은 항목이 두 번 나오거나 누락될 수 있다. 팀과 조율 중이다.