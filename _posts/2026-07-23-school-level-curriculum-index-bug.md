---
title: "중1 2학기 6단원이 초1 단원으로 잡힌 진짜 이유"
date: 2026-07-23 17:00:00 +0900
categories: [Backend]
tags: [debugging, domain-modeling, sql, dataclass]
---

추천 결과를 눈으로 훑다가 걸린 게 있다. 사용자가 지목한 건 중1 2학기 6단원인데, 시스템이 물어다 준 건 초1 단원. 처음엔 데이터가 오염됐나 싶었다. 콘텐츠 메타 테이블을 직접 열어봤더니 row는 멀쩡했다. `school_level` 값도 다 붙어 있었다. 문제는 데이터가 아니었다.

## 함수 시그니처가 이미 답을 반쯤 정해놨다

커리큘럼 단원을 찾는 함수는 이렇게 생겨 있었다.

```python
def find_unit_by_grade_semester_chapter(
    grade: int,
    semester: int,
    chapter: int,
) -> CurriculumUnit:
    ...
```

세 개의 정수. `grade=1, semester=2, chapter=6`.

머릿속으로 자연스러워 보인다는 게 함정이었다. 국내 교육과정에서 이 세 값의 조합은 **초등 1학년에도, 중학교 1학년에도, 고등 1학년에도** 각각 존재한다. 학교급을 안 넣는 순간, DB 인덱스가 먼저 만나는 row가 정답이 된다. 그날 초1이 먼저 잡힌 건 그냥 운이었다.

## 스키마는 알고 있었는데 코드가 몰랐다

더 어이없는 건 테이블에는 이미 컬럼이 있었다는 거다.

```sql
CREATE TABLE curriculum_units (
    id           INTEGER PRIMARY KEY,
    school_level TEXT NOT NULL,   -- 'elementary' | 'middle' | 'high'
    grade        INTEGER NOT NULL,
    semester     INTEGER NOT NULL,
    unit_order   INTEGER NOT NULL,
    title        TEXT NOT NULL
);
```

DDL 짤 때 누군가는 `school_level`이 유일성의 일부라는 걸 알고 있었다. 그런데 Repository 쪽 dataclass에서 이 필드가 빠져 있었다.

```python
@dataclass
class CurriculumUnit:
    id: int
    grade: int
    semester: int
    unit_order: int
    title: str
```

dataclass가 이러니 SELECT 문도 자연스럽게 이 모양이 됐다.

```sql
SELECT id, grade, semester, unit_order, title
FROM curriculum_units
WHERE grade = ? AND semester = ? AND unit_order = ?
```

WHERE에도 없고 SELECT에도 없다. 초등·중등·고등 세 세계가 같은 키 공간에서 겹쳐 있는데, 조회 코드는 그걸 구분할 언어 자체가 없었다. 결과는 무음(silent). 에러도 안 나고, 그냥 엉뚱한 단원을 잘 리턴한다. 이런 버그가 제일 무섭다.

## 고친 건 세 곳, 하지만 사실은 한 곳

수정 자체는 지루할 만큼 단순했다. 조회 키를 `(school_level, grade, semester, unit_order)` 네 개짜리 복합 키로 정의하고, dataclass·함수 시그니처·SQL을 한 번에 맞췄다.

```python
@dataclass
class CurriculumUnit:
    id: int
    school_level: str
    grade: int
    semester: int
    unit_order: int
    title: str


def find_unit(
    school_level: str,
    grade: int,
    semester: int,
    unit_order: int,
) -> CurriculumUnit:
    ...
```

```sql
SELECT id, school_level, grade, semester, unit_order, title
FROM curriculum_units
WHERE school_level = ?
  AND grade        = ?
  AND semester     = ?
  AND unit_order   = ?
```

세 곳을 고쳤지만 사실 고친 건 한 곳이다. **이 도메인에서 단원의 식별자가 뭔지에 대한 답**. 그동안은 그 답이 코드 어디에도 안 적혀 있었다. dataclass에도, 시그니처에도, WHERE 절에도. 세 군데가 서로를 참고해가며 잘못된 합의를 유지하고 있었던 셈이다.

## 테스트를 먼저 썼다

한 번 이런 걸 겪으면, 다음번엔 다른 방식으로 다시 튀어나온다. 회귀 테스트를 먼저 심어놓고 고쳤다.

```python
def test_same_grade_semester_chapter_differ_by_school_level():
    elem = find_unit('elementary', 1, 2, 6)
    mid  = find_unit('middle',     1, 2, 6)
    assert elem.id != mid.id
    assert elem.school_level == 'elementary'
    assert mid.school_level  == 'middle'
```

수정 전에는 두 호출이 같은 row를 뱉었다. 수정 후 갈라졌다. 이 테스트가 하는 일은 사실 검증보다 **문서화**에 가깝다. "이 도메인에서 단원은 학교급까지 포함해야 유일하다"를 코드로 못박아 두는 것.

## 남은 찜찜함

이번 건 커리큘럼 쪽이었지만, 같은 냄새가 나는 곳이 다른 도메인에도 있을 것 같다. 예를 들면 직무·NCS·세부직무 매핑처럼 상위 분류가 있어야 유일해지는 계층 구조들. 실제 추천 엔진 쪽 상수 파일을 보면 `JOB_TO_NCS`, `SUB_BY_JOB`, `SUB_TO_KEYWORDS`처럼 계층을 타고 내려가는 매핑이 여러 겹 쌓여 있고, 이 사이에서 키가 상위 분류 없이 단독으로 조회되는 지점이 있는지 아직 다 훑지 못했다.

교훈으로 마무리하고 싶지 않다. 그냥 이번엔 학교급이 빠져 있었고, 다음번엔 다른 상위 분류자가 어딘가 빠져 있을 거다. 그걸 찾는 건 다음 주 할 일로 남겨둔다.