---
title: "중1 2학기 6단원이 초1 단원으로 잡힌 이유"
date: 2026-07-24 17:00:00 +0900
categories: [Backend]
tags: [database, schema-design, debugging, python]
---

"중1 2학기 6단원"을 달라고 했더니 초1 데이터가 돌아왔다.

쿼리는 에러 없이 통과했다. DB에도 중1 행은 멀쩡히 들어가 있었다. 그런데 결과는 초등학교 것. 로그를 보고 처음엔 데이터 적재 파이프라인부터 의심했다. 아니었다. 데이터는 죄가 없었다.

## 재현

문제의 조회는 이 정도 모양이었다.

```python
async def find_unit_by_grade_semester_chapter(grade, semester, chapter):
    return await db.fetch_one(
        "SELECT * FROM curriculum_units "
        "WHERE grade = :grade AND semester = :semester AND unit_order = :chapter",
        {"grade": grade, "semester": semester, "chapter": chapter}
    )
```

`grade=1, semester=2, unit_order=6`. 이 세 값으로 초1 2학기 6단원도 잡히고, 중1 2학기 6단원도 잡힌다. 한국 수학 교육과정은 학교급이 바뀔 때마다 학년이 다시 1부터 시작하니까. 초1이 먼저 삽입돼 있었으니 초1이 이겼다. `fetch_one` 은 순서를 약속하지 않는데도, 실제 결과는 삽입 순서에 끌려다니고 있었다.

버그라기보다는 **식별자를 하나 빠뜨린 채로 유일성을 기대한 것**에 가까웠다.

## 스키마는 알고 있었다

```sql
CREATE TABLE curriculum_units (
    id INTEGER PRIMARY KEY,
    school_level TEXT NOT NULL,   -- '초등' | '중등' | '고등'
    grade INTEGER NOT NULL,
    semester INTEGER NOT NULL,
    unit_order INTEGER NOT NULL,
    unit_name TEXT NOT NULL
);
```

`school_level` 은 처음부터 있었다. 문제는 애플리케이션 쪽이었다. dataclass 정의부터가 이랬다.

```python
@dataclass
class CurriculumUnit:
    id: int
    grade: int
    semester: int
    unit_order: int
    unit_name: str
    # school_level 이 없다
```

DB는 "초·중·고를 나눠서 저장한다"고 말하고 있는데, 코드는 그 말을 안 듣고 있었다. SELECT 목록에도 없고, WHERE 조건에도 없고, 모델에도 없다. 그러니 앱 입장에서 초1과 중1은 같은 행이었다.

이걸 발견하고 나서 잠깐 멍했다. 스키마 설계 때 `school_level` 을 넣은 건 나였다. 넣어놓고 그걸 안 쓴 것도 나였다. 두 세계가 어긋난 채로 몇 주를 굴러갔다.

## 4-tuple로 되돌리기

고치는 건 오래 걸리지 않았다. `(grade, semester, unit_order)` 로 좁혀서 유일하다고 착각했던 걸, `(school_level, grade, semester, unit_order)` 로 되돌리기만 하면 됐다.

```python
@dataclass
class CurriculumUnit:
    id: int
    school_level: str
    grade: int
    semester: int
    unit_order: int
    unit_name: str


async def find_unit(school_level: str, grade: int, semester: int, chapter: int):
    return await db.fetch_one(
        "SELECT * FROM curriculum_units "
        "WHERE school_level = :school_level "
        "  AND grade = :grade "
        "  AND semester = :semester "
        "  AND unit_order = :chapter",
        {
            "school_level": school_level,
            "grade": grade,
            "semester": semester,
            "chapter": chapter,
        },
    )
```

시그니처가 바뀌니 호출부에서 `school_level` 을 넘겨야 하는 게 강제된다. 이게 지금 상태에서 얻은 것 중 제일 큰 이득이다. 컴파일러가 없는 언어에서 "잊고 안 넣었네"를 막는 방법은 결국 함수 시그니처에 박아두는 것뿐이다.

인덱스도 같이 걸었다.

```sql
CREATE INDEX idx_curriculum_lookup
ON curriculum_units (school_level, grade, semester, unit_order);
```

원래는 UNIQUE로 갈까 잠깐 고민했는데, 교과서 개편 시기에 같은 (학교급, 학년, 학기, 단원번호) 가 잠깐 겹쳐 존재하는 케이스가 있을 수도 있어서 일단 일반 인덱스로 뒀다. 이 부분은 아직 결정을 미뤄놓았다.

## 남은 것

테스트 하나는 넣어뒀다. "중1 2학기 6단원을 조회하면 초1이 오면 안 된다" 는, 이 상황이 아니면 아무도 안 쓸 것 같은 회귀 테스트다.

```python
def test_middle_school_unit_not_confused_with_elementary():
    unit = find_unit_sync("중등", grade=1, semester=2, chapter=6)
    assert unit.school_level == "중등"
```

정작 무서운 건, 이 프로젝트에 `curriculum_units` 처럼 "번호가 카테고리 안에서만 유일한" 테이블이 이거 하나뿐이 아니라는 것이다. 학교급 말고도 교과서 출판사, 개정 연도, 이런 축이 더 붙는다. 지금 이 4-tuple 로도 충분한지, 개정 연도가 섞이는 순간 또 같은 실수를 하지는 않을지 — 이건 다음에 스키마를 다시 훑을 때 봐야 할 몫으로 남겨뒀다.