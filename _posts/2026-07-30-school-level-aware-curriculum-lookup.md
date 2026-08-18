---
title: "중1 2학기 6단원이 초1 단원으로 잡힌 작은 버그"
date: 2026-07-30 17:00:00 +0900
categories: [Backend]
tags: [db-schema, composite-key, debugging]
---

추천 결과를 확인하다 이상한 걸 봤다. 중학교 1학년 2학기 6단원을 입력했는데, 화면에 뜬 단원명은 초등학교 단원이었다.

숫자는 다 맞았다. `grade=1, semester=2, chapter=6`. 셋 다 요청한 대로 들어갔고, 반환된 행도 정확히 그 세 값을 가지고 있었다. 문제는 그 세 값을 만족하는 행이 DB에 두 개였다는 것이다. 초등 1학년 2학기 6단원, 그리고 중학교 1학년 2학기 6단원.

## 스키마는 알고 있었다

먼저 DB부터 봤다. `school_level` 컬럼은 이미 있었다.

```sql
CREATE TABLE curriculum_units (
    id INTEGER PRIMARY KEY,
    school_level TEXT NOT NULL,  -- 'elementary' | 'middle' | 'high'
    grade INTEGER NOT NULL,
    semester INTEGER NOT NULL,
    unit_order INTEGER NOT NULL,
    title TEXT NOT NULL
);
```

설계 단계에서 누군가는 학교급을 구분해야 한다는 걸 알고 있었던 것이다. 컬럼도 `NOT NULL`로 잡혀 있었고, 데이터도 정상적으로 채워져 있었다. 그러니까 이건 데이터 문제가 아니었다.

DB에는 있는데 코드에는 없는 상태였다.

## repository 레이어에서 사라진 컬럼

Python 쪽으로 넘어가 dataclass부터 확인했다.

```python
@dataclass
class CurriculumUnit:
    id: int
    grade: int
    semester: int
    unit_order: int
    title: str
    # school_level 없음
```

SELECT 쿼리도 마찬가지였다.

```sql
SELECT id, grade, semester, unit_order, title
FROM curriculum_units
WHERE grade = ? AND semester = ? AND unit_order = ?
```

`school_level`은 세 곳 다 빠져 있었다. dataclass에도, SELECT 컬럼 목록에도, WHERE 조건에도. 조회 함수 시그니처도 자연스럽게 이 형태였다.

```python
def find_unit_by_grade_semester_chapter(
    grade: int,
    semester: int,
    chapter: int,
) -> CurriculumUnit | None:
    ...
```

세 개의 정수만 받아서 세 개의 정수만 비교한다. `ORDER BY`가 명시되지 않았으니 어떤 게 먼저 나올지는 사실 보장이 없는데, 우리 쪽에선 `id` 오름차순으로 먹혀서 항상 초등 쪽이 먼저 튀어나오고 있었다. 그래서 "가끔 틀린다"가 아니라 "중학교 케이스는 늘 초등으로 잡힌다"였다. 오히려 이게 다행이었다. 간헐적이었으면 재현하다가 몇 시간 더 썼을 것이다.

## 세 곳을 같이 바꿔야 했다

한 곳만 바꾸면 안 된다는 점이 이 수정의 성가신 지점이었다. dataclass에 필드만 추가하면 SELECT가 안 채워주니 `None`이 되고, SELECT만 늘리면 dataclass가 안 받는다. WHERE만 조이면 함수를 호출하는 쪽이 아직 인자를 안 넘긴다.

```python
@dataclass
class CurriculumUnit:
    id: int
    school_level: str
    grade: int
    semester: int
    unit_order: int
    title: str
```

```sql
SELECT id, school_level, grade, semester, unit_order, title
FROM curriculum_units
WHERE school_level = ? AND grade = ? AND semester = ? AND unit_order = ?
```

```python
def find_unit_by_grade_semester_chapter(
    school_level: str,
    grade: int,
    semester: int,
    chapter: int,
) -> CurriculumUnit | None:
    ...
```

역방향 조회용 인메모리 딕셔너리도 같이 손봐야 했다. 단원 → id를 다시 뒤집는 캐시가 있었는데, 이쪽 키가 `(grade, semester, unit_order)` 세 튜플이었다. 여기서도 중학교 6단원이 초등 6단원으로 덮어써지고 있었을 것이다. 키를 네 튜플로 늘렸다.

```python
# 변경 전
(grade, semester, unit_order) -> CurriculumUnit
# 변경 후
(school_level, grade, semester, unit_order) -> CurriculumUnit
```

## 테스트 케이스로 남겼다

수정 후에 회귀 테스트를 하나 붙였다. 같은 `grade/semester/chapter` 조합에 `school_level`만 바꿔 넣었을 때, 서로 다른 단원이 나와야 한다는 케이스.

수정 전엔 두 케이스 모두 초등 단원을 반환했고, 수정 후엔 정확히 갈라졌다. 이 테스트를 남긴 이유는, 앞으로 누가 `school_level`을 다시 빼먹는 리팩터를 하더라도 이 케이스에서 걸리라는 뜻이다.

## 남은 것

이 도메인엔 비슷한 함정이 더 있을 것 같다. 계층이 있는 식별자를 세 값만으로 잡는 함수가 다른 데도 남아 있을 가능성. `locale`, `tenant_id`, `region` 같은 상위 구분자가 슬며시 빠져 있는 곳. 이번 건은 화면에서 단원명이 대놓고 초등이라 눈에 띈 거고, 안 띄는 곳에서 조용히 틀리고 있는 조회가 더 있을 수 있다.

컬럼이 스키마엔 있는데 dataclass엔 없는 경우를 정적으로 잡아낼 방법이 있을까 — 이건 아직 안 봤다. 다음에 볼 것.