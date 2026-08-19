---
title: "테이블명 하나 바꾸기 전에 DB로 먼저 증명한 이유"
date: 2026-08-19 17:00:00 +0900
categories: [Backend]
tags: [MySQL, debugging, schema-migration, SQL]
---

API가 조용히 깨질 때가 제일 무섭다.

에러를 던지면 오히려 낫다. 발견이라도 할 수 있으니까. 그런데 이번엔 응답은 나왔다. 200도 반환됐다. 다만 특정 조회에서 태그 데이터가 빠져 있었다.

추적을 시작했다. 원인은 얼마 전 완료한 테이블 rename이었다. `content_project_tags`를 `content_tags`로 이름을 바꿨는데, 그 이름을 참조하는 쿼리 하나가 업데이트되지 않은 채로 남아 있었다.

```sql
LEFT JOIN content_project_tags t ON t.content_id = c.id
```

이 한 줄이었다. DB에 없는 테이블을 JOIN하고 있으니, 드라이버에 따라 에러가 나거나 빈 결과를 반환하거나. 이번엔 후자였다. LEFT JOIN이어서 쿼리 자체는 성공했고, 태그만 조용히 `NULL`로 돌아왔다. 로그에 아무것도 안 찍혔다.

---

고치기 전에 먼저 확인하기로 했다.

```bash
python scripts/check_schema.py
```

이 스크립트는 현재 DB 스키마를 읽어 테이블 목록과 컬럼 구조를 출력한다. 여기서 두 가지를 따로 확인했다.

**첫째, 옛 테이블이 정말로 없는가.**

```sql
SHOW TABLES LIKE 'content_project_tags';
-- Empty set
```

없었다. rename이 완료됐고 이전 이름은 사라진 상태.

**둘째, 새 테이블이 실제로 데이터를 반환하는가.**

```sql
SELECT * FROM content_tags LIMIT 5;
```

데이터가 있었다. 컬럼 구조도 맞았다.

이 두 단계를 굳이 나눈 이유가 있다. "rename됐으니 당연히 새 테이블이 있겠지"라고 넘겼다가, 마이그레이션 도중 이름이 엇갈리거나 스키마가 다른 환경을 바라보는 경우를 전에 한 번 겪었다. 그래서 직접 확인한다.

---

세 번째 확인은 LEFT JOIN 수정 후 동작 검증이었다.

LEFT JOIN이니까 `content_tags`에 매칭되는 행이 없는 콘텐츠도 결과에 살아 있어야 한다. 태그가 없다고 레코드 자체가 사라지면 안 된다.

```sql
SELECT c.id, t.tag_name
FROM content c
LEFT JOIN content_tags t ON t.content_id = c.id
WHERE c.id IN (/* 태그 미등록 콘텐츠 ID 샘플 */);
```

태그가 없는 콘텐츠는 `tag_name = NULL`로 행이 살아 있었다. 정상이다.

이걸 빼먹으면, JOIN 테이블명은 고쳤는데 "태그 없는 콘텐츠가 목록에서 통째로 사라졌다"는 버그를 새로 심을 수 있다. LEFT JOIN의 null 허용 의미를 명시적으로 쿼리로 확인해두면 이 실수를 막는다.

---

실제 수정은 한 줄이었다.

```sql
-- before
LEFT JOIN content_project_tags t ON t.content_id = c.id

-- after
LEFT JOIN content_tags t ON t.content_id = c.id
```

이게 전부다.

수정 이후 같은 작업 흐름에서 페이지네이션 응답 포맷도 정비했다. `totalCount / list` 구조로 통일하고, status 필드는 API 스펙에서 뺐다. 거기서 랜덤 추천 API에 페이지네이션을 붙이는 문제가 나왔다. 페이지를 넘길 때마다 랜덤이 새로 섞이면 중복·누락이 생긴다. seed를 고정하면 "랜덤"이라는 의미가 흐려진다. 지금은 응답 포맷만 맞추고 랜덤 로직은 건드리지 않은 상태로 뒀다.

이게 언제 다시 수면 위로 올라올지는 모르겠다.