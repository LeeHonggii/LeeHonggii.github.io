---
title: "중1 2학기 6단원이 초1 단원으로 잡힌 이유"
date: 2026-06-22 17:00:00 +0900
categories: [Backend]
tags: [curriculum-mapping, debugging, dataclass, database]
---

# 도입 (왜 이 글)

어느 날 추천 파이프라인 로그를 보다가 이상한 걸 발견했다. 중학교 1학년 2학기 6단원을 조회했는데, 반환된 단원명이 영락없이 초등학교 단원이었다. DB에 데이터가 잘못 들어간 건가? 아니었다. 코드가 처음부터 학교급(school level)을 무시하고 단원을 찾고 있었다.

## 충돌의 구조: 같은 번호, 다른 단원

교육과정 DB를 보면 이런 행들이 공존한다.

| school_level | grade | semester | unit_order | unit_name |
|---|---|---|---|---|
| elementary | 1 | 2 | 6 | 덧셈과 뺄셈 |
| middle | 1 | 2 | 6 | 일차방정식 |
| high | 1 | 2 | 6 | 집합과 명제 |

`grade=1, semester=2, unit_order=6`이라는 키 조합은 세 레코드에서 동시에 성립한다. `school_level`이 없으면 DB는 그냥 먼저 만나는 행을 돌려준다. 운이 나쁘면 초등 단원이 중등 질의의 결과가 된다.

## 버그의 진짜 원인: dataclass와 SELECT의 누락

DB 스키마에는 `school_level` 컬럼이 분명히 있었다. 문제는 두 군데서 동시에 빠져 있었다.

**1. Repository dataclass**

```python
@dataclass
class CurriculumUnit:
    grade: int
    semester: int
    unit_order: int
    unit_name: str
    # school_level 없음 ← 여기서부터 앱에는 존재하지 않는 필드
```

**2. SELECT 쿼리**

```sql
SELECT grade, semester, unit_order, unit_name
FROM curriculum_units
WHERE grade = ? AND semester = ? AND unit_order = ?
-- school_level 조건 없음
```

데이터는 DB에 있었지만, 앱 레이어에 올라오는 순간 사라졌다. `CurriculumUnit.school_level`을 참조하는 코드는 `AttributeError`가 날 것이므로 아무도 쓰지 않는 필드가 됐고, 그 결과 lookup key에서도 자연스럽게 빠졌다.

## 수정: lookup key를 4-tuple로 바꾸기

수정은 세 단계였다.

**dataclass에 필드 추가**

```python
@dataclass
class CurriculumUnit:
    school_level: str   # "elementary" | "middle" | "high"
    grade: int
    semester: int
    unit_order: int
    unit_name: str
```

**SELECT에 컬럼·조건 추가**

```sql
SELECT school_level, grade, semester, unit_order, unit_name
FROM curriculum_units
WHERE school_level = ?
  AND grade = ?
  AND semester = ?
  AND unit_order = ?
```

**호출부 시그니처 변경**

```python
# 변경 전
find_unit_by_grade_semester_chapter(grade, semester, chapter)

# 변경 후
find_unit_by_grade_semester_chapter(school_level, grade, semester, chapter)
```

호출부가 `school_level`을 모르면 그 위 계층에서 넘겨줘야 한다. 이 변경이 연쇄적으로 올라가면서, 파이프라인 입력 단계부터 학교급을 명시적으로 다뤄야 한다는 사실이 드러났다. 숨어 있던 암묵적 가정이 수면 위로 올라온 것이다.

## 회귀 테스트로 못 박기

수정 후 같은 버그가 재발하지 않도록 케이스를 고정했다.

```python
def test_middle_school_unit_does_not_return_elementary():
    unit = find_unit_by_grade_semester_chapter(
        school_level="middle", grade=1, semester=2, chapter=6
    )
    assert unit.unit_name == "일차방정식"
    assert unit.school_level == "middle"
```

테스트 이름에 의도를 박아두면, 나중에 누군가 `school_level` 조건을 지우더라도 즉시 실패로 알 수 있다.

## 마치며

이 버그의 교훈은 단순하다. **DB 컬럼이 있어도 dataclass와 SELECT에서 빠지면 앱에는 없는 데이터다.** 교육과정처럼 계층 구조가 있는 도메인에서는 `(grade, semester, unit_order)`만으로 단원을 특정할 수 없다. lookup key를 `(school_level, grade, semester, unit_order)`로 설계하는 것이 맞다. 스키마를 처음 설계할 때 이 4-tuple을 복합 유니크 키로 걸어뒀다면 쿼리를 짜는 시점에 바로 알 수 있었을 것이다. 다음 번엔 그렇게 하려 한다.