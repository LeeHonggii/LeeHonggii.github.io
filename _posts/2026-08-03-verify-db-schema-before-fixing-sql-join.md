---
title: "SQL 한 줄 고치기 전에 DB로 먼저 증명한 이야기"
date: 2026-08-03 17:00:00 +0900
categories: [Backend]
tags: [sql, debugging, schema-migration, join]
---

카테고리 조회 API가 이상하게 비어있었다. 에러는 없었다. 그냥 결과가 없었다.

이게 제일 성가신 종류의 버그다. 500이 떨어지면 스택트레이스를 따라가면 되는데, 200에 빈 배열이 오면 어디서부터 봐야 할지 감이 없다. 응답 형식은 맞다. 쿼리도 돈다. 단지 아무것도 안 나올 뿐이다.

## 코드부터 열지 않은 이유

원인 후보는 금방 좁혀졌다. 얼마 전에 테이블 rename이 있었다는 걸 기억하고 있었다. 마이그레이션은 통과했고 ORM 모델은 새 이름으로 갈아탔지만, 그 API 하나는 raw SQL을 직접 쓰고 있었다. JOIN 절 어딘가에 옛 이름이 남아있을 거라는 가설.

여기서 그냥 편집기를 열고 옛 이름을 찾아 바꾸면 30초짜리 작업이다. 나도 처음엔 그럴 뻔했다. 근데 손이 잠깐 멈춘 이유는, **틀린 이름도 그럴싸하다는 걸** 알기 때문이었다. `category`, `categories`, `canvas_category`, `content_category` — 네 개 다 코드에 나올 법한 이름이고, 네 개 중 어느 게 살아있는 실제 테이블인지는 코드가 알려주지 않는다. 마이그레이션 파일을 거슬러 올라가서 재구성할 수도 있지만, 그건 결국 추측을 강화하는 일이다.

DB한테 직접 물어보면 된다.

## 검증 스크립트

`scripts/` 밑에 확인용 파일을 하나 만들었다. 지우려고 만든 게 아니라, 남기려고 만든 것이다.

```python
import sqlite3

conn = sqlite3.connect("app.db")
cur = conn.cursor()

cur.execute("SELECT name FROM sqlite_master WHERE type='table';")
print("tables:", cur.fetchall())

cur.execute("PRAGMA table_info(content_category);")
print("columns:", cur.fetchall())

cur.execute("""
    SELECT c.id, c.name, t.name AS tag_name
    FROM content_category c
    LEFT JOIN content_tag t ON t.category_id = c.id
    LIMIT 10;
""")
for row in cur.fetchall():
    print(row)

conn.close()
```

결과가 나오자 두 가지가 한 번에 정리됐다. 실제 테이블 이름은 rename 이후의 새 이름이었고, 코드에 박혀있던 건 완전히 다른 옛 이름이었다. "비슷하니까 그냥 s만 붙이면 되겠지" 같은 한 글자 차이가 아니라, 접두어 자체가 달랐다. 코드만 보고 고쳤다면 새 옛날 이름을 하나 더 만들어냈을 가능성이 충분히 있다.

## null이 섞여 나왔을 때

출력을 보다가 한 번 더 손이 멈췄다. 몇 행에서 `tag_name`이 `None`이었다.

```
(1, '공개', '인기')
(2, '공개', '신규')
(3, '비공개', None)
```

순간 "이것도 문제인가" 싶었다. 여기서 서두르면 두 번째 버그를 새로 만들었을 것이다. `LEFT JOIN`을 `INNER JOIN`으로 바꿔서 null을 없애는 식으로. 그런데 이 API의 기획 의도는 **태그가 붙지 않은 카테고리도 목록에 나와야 한다**는 쪽이었다. 그러니까 null은 버그가 아니라 정확히 요구사항이 요구하는 모양이었다. `LEFT JOIN`이 자기 일을 하고 있었을 뿐이다.

한 발짝만 늦게 판단하면 되는 걸 자꾸 잊는다.

## 고친 건 한 줄

확인이 끝난 뒤에 실제 수정은 시시했다.

```sql
-- before
LEFT JOIN old_tag t ON t.category_id = c.id

-- after
LEFT JOIN content_tag t ON t.category_id = c.id
```

이걸 위해 10분을 더 썼다. 처음엔 낭비처럼 느껴졌는데, 스크립트를 지우지 않고 `scripts/`에 남긴 뒤로는 생각이 바뀌었다. 다음번에 같은 API를 리뷰하거나 배포 전에 스모크를 돌리고 싶을 때, 이 파일을 실행하면 몇 초 안에 "쿼리가 실제로 살아있는 스키마 위에서 도는가"가 확인된다. 처음 만들 땐 10분, 이후엔 10초.

## 남은 것

이 문제 하나만 놓고 보면 깔끔한 이야기인데, 사실 걸리는 건 따로 있다. **rename 이후에 raw SQL이 다 갱신됐는지 자동으로 확인할 방법이 없다.** ORM은 모델을 바꾸면 컴파일러나 타입 체커가 잡아주는 편인데, 문자열 안에 들어있는 테이블명은 아무도 안 잡아준다. 이번엔 우연히 이 API 하나만 raw였고, 우연히 사용자 리포트가 들어왔다.

CI에서 raw SQL을 골라내 실제 DB(테스트용 복제본)에 EXPLAIN이라도 돌려보는 훅을 붙일 수 있지 않을까 생각 중이다. 아직 안 해봤다. 다음 rename이 들어올 때까진 미뤄질 것 같긴 하다.