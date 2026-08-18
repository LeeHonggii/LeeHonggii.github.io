---
title: "중1 2학기 6단원이 초1 단원으로 잡힌 이유: curriculum lookup에 school_level이 빠져 있었다"
date: 2026-07-28 17:00:00 +0900
categories: [Backend]
tags: [debugging, schema, natural-key, silent-bug]
---

로그를 뒤지다가 눈에 걸린 줄이 하나 있었다. 중학교 1학년 2학기 6단원을 넘겼는데, 응답에 실려온 단원 이름이 통계가 아니었다. 초등학교 1학년 어느 단원이었다.

예외는 없었다. 쿼리도 성공했고, 응답 코드도 200이었다. 그냥 조용히 틀린 값을 돌려주고 있었다.

## 처음 의심한 건 데이터였다

당연히 DB부터 열어봤다. `curriculum_units` 테이블에 이상한 레코드가 섞였나. `unit_name='통계'`로 조회해보니 멀쩡히 있었다. `school_level='middle'`, `grade=1`, `semester=2`, `unit_order=6`. 지워지지도, 중복되지도 않았다.

그럼 왜 못 가져오나. 조회 함수를 열었다.

```python
def find_unit_by_grade_semester_chapter(grade: int, semester: int, chapter: int) -> CurriculumUnit:
    row = db.query(
        "SELECT * FROM curriculum_units "
        "WHERE grade=? AND semester=? AND unit_order=?",
        (grade, semester, chapter),
    ).fetchone()
    return CurriculumUnit(**row)
```

세 개다. `grade`, `semester`, `unit_order`. `school_level`이 없다.

문제가 여기서 명확해졌다. "1학년 2학기 6단원"은 초등에도 있고, 중등에도 있고, 어쩌면 고등에도 있다. 학년 숫자는 학교급 안에서만 유일하다. 그런데 조회 키에는 학교급이 빠져 있었다. DB는 시키는 대로 조건을 만족하는 첫 행을 돌려줬고, 그게 마침 초1 레코드였다.

## 스키마에는 있었다. 코드에만 없었다

이상한 건 DB 쪽이었다. 컬럼이 있었다.

```sql
CREATE TABLE curriculum_units (
    id            INTEGER PRIMARY KEY,
    school_level  TEXT,        -- 'elementary' | 'middle' | 'high'
    grade         INTEGER,
    semester      INTEGER,
    unit_order    INTEGER,
    unit_name     TEXT,
    ...
);
```

INSERT 시점에는 채워지고 있었다. `school_level='middle'`인 행도, `'elementary'`인 행도 잘 들어갔다. 다만 아무도 그 값을 **읽지** 않았다.

repository 계층의 dataclass도 열어봤다.

```python
@dataclass
class CurriculumUnit:
    id: int
    grade: int
    semester: int
    unit_order: int
    unit_name: str
```

역시 없다. `SELECT *`로 뽑아도 `school_level` 컬럼은 dict 언팩 과정에서 조용히 버려지고 있었다 — 아니, 정확히는 `**row`로 넘길 때 dataclass에 없는 키가 있으면 `TypeError`가 나야 정상인데, 실제로는 row를 tuple 위치 매핑으로 넘기는 어댑터를 거치고 있어서 컬럼 하나가 슬쩍 밀리기만 했다. 그래서 `school_level` 자리에 있어야 할 값이 다른 필드로 들어가거나, `SELECT`를 명시적으로 열거한 다른 경로에선 그냥 사라지거나 했다.

버그가 시끄럽지 않았던 이유가 여기 있다. 어디에서도 예외가 안 나게 되어 있었다. 컬럼 하나가 없다는 사실을 코드가 아무데서도 확인하지 않았다.

## 재현부터 고정했다

수정에 손을 대기 전에, "이 케이스는 통계가 나와야 한다"를 테스트로 박아뒀다. 안 그러면 고친 뒤에도 진짜 고쳐진 건지 알 방법이 없다.

```python
def test_middle_1_2_6_returns_statistics_unit():
    unit = find_unit_by_grade_semester_chapter(
        school_level="middle", grade=1, semester=2, chapter=6,
    )
    assert unit.school_level == "middle"
    assert unit.unit_name == "통계"

def test_elementary_1_2_6_still_works():
    unit = find_unit_by_grade_semester_chapter(
        school_level="elementary", grade=1, semester=2, chapter=6,
    )
    assert unit.school_level == "elementary"
```

두 번째 테스트가 있는 이유는, 초등 쪽 조회가 지금까지 (우연히) 잘 되고 있었기 때문이다. 이건 우연이 맞는 방향으로 작용한 케이스라, 고치면서 깨질 수 있다.

## 자연키를 4-튜플로 다시 정의

수정 자체는 크지 않았다. dataclass에 필드 하나 추가하고, 조회 함수 시그니처에 인자 하나 붙이고, WHERE에 컬럼 하나 얹었다.

```python
@dataclass
class CurriculumUnit:
    id: int
    school_level: str
    grade: int
    semester: int
    unit_order: int
    unit_name: str


def find_unit_by_grade_semester_chapter(
    school_level: str, grade: int, semester: int, chapter: int,
) -> CurriculumUnit:
    row = db.query(
        "SELECT id, school_level, grade, semester, unit_order, unit_name "
        "FROM curriculum_units "
        "WHERE school_level=? AND grade=? AND semester=? AND unit_order=?",
        (school_level, grade, semester, chapter),
    ).fetchone()
    return CurriculumUnit(**row)
```

`SELECT *`도 없앴다. 컬럼을 명시적으로 열거하면, 다음에 누가 스키마에 컬럼을 추가했을 때 자동으로 dataclass에 흘러 들어가는 일이 없어진다. 컬럼과 dataclass는 손으로 맞춰야 한다 — 그게 오히려 안전하다.

인덱스도 새로 걸었다. 기존 인덱스는 `(grade, semester, unit_order)`였는데, `school_level`이 앞에 붙는 게 자연키 순서에 맞다.

```sql
CREATE INDEX idx_curriculum_lookup
ON curriculum_units (school_level, grade, semester, unit_order);
```

## 호출부가 더 오래 걸렸다

정작 시간을 잡아먹은 건 함수 안이 아니라 함수 **바깥**이었다. `find_unit_by_grade_semester_chapter(1, 2, 6)`으로 부르던 자리가 코드베이스 곳곳에 있었고, 어떤 데는 학교급 정보를 아예 안 들고 있었다. 요청 payload에서 `school_level`이 들어오지 않는 라우트도 있었고, 시리즈 이어보기용으로 이전 콘텐츠의 grade만 뽑아서 쓰던 자리도 있었다.

이런 자리를 하나씩 훑으면서 두 가지를 판단해야 했다:

- 호출 컨텍스트에서 `school_level`을 **알고 있는데 안 넘기고 있었나** — 그러면 그냥 붙이면 된다.
- 애초에 그 컨텍스트가 학교급을 모르나 — 그러면 어디까지 거슬러 올라가야 학교급이 잡히는지 추적해야 한다.

두 번째 경우가 더 아팠다. 결국 학교급을 조회 파라미터가 아니라 세션/사용자 컨텍스트에서 함께 흘려보내는 쪽으로 손을 봤는데, 이 작업은 이 글의 주제를 벗어난다.

## 아직 남은 것

고등학교 커리큘럼 쪽은 아직 안 봤다. `school_level='high'` 레코드가 있긴 한데, 조회가 실제로 안 걸리는 자리에만 있는지, 아니면 같은 종류의 실수가 다른 함수에 또 있는지 확인이 필요하다. 이번 버그가 유일한 사례라고 믿을 근거가 아직 없다.

그리고 dataclass ↔ 스키마 어긋남을 CI에서 잡을 방법도 생각만 해두고 아직 안 붙였다. `information_schema`를 읽어서 dataclass 필드와 비교하는 정도면 될 것 같은데, 실제 도입해봐야 안다. 컬럼 이름이 바뀌는 마이그레이션이 얽히면 그렇게 단순하진 않을 것이다.

버그 자체는 5줄로 끝났는데, 이 5줄이 얼마나 오래 살아 있었는지가 더 신경 쓰인다. 예외를 던지지 않는 버그는 나이를 안 먹는다.