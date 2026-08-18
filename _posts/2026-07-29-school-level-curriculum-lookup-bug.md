---
title: "중1 2학기 6단원이 왜 초등 단원으로 잡혔을까"
date: 2026-07-29 17:00:00 +0900
categories: [Backend]
tags: [식별자설계, 데이터모델링, 디버깅]
---

단원 추천 로직을 붙이고 나서 손으로 몇 개 찍어보다가, "중1 2학기 6단원"이 통계 단원이 아니라 초등학교 어떤 단원으로 잡히는 걸 봤다. 데이터가 비어서가 아니었다. 중등 커리큘럼은 분명히 들어와 있었다. 코드가 다른 행을 집어오고 있었다.

원인 자체는 30분도 안 걸려서 찾았다. 그런데 "왜 이 버그가 이 자리에 있을 수 있었나"를 이해하는 건 그보다 오래 걸렸고, 지금도 완전히 정리됐다고 말하기 어렵다.

## 처음 본 조회 코드

문제가 된 부분은 대충 이런 모양이었다.

```python
def find_unit(grade: int, semester: int, chapter: int) -> Unit:
    return db.query(
        "SELECT * FROM units WHERE grade=? AND semester=? AND chapter=?",
        (grade, semester, chapter),
    ).fetchone()
```

`grade=1, semester=2, chapter=6` 이 들어오면 초1 2학기 6단원과 중1 2학기 6단원이 둘 다 조건을 만족한다. 어느 게 먼저 나올지는 그날의 인덱스와 삽입 순서가 정한다. 운이 나쁘면 계속 초등 쪽이 나온다.

DB 스키마에는 `school_level` 컬럼이 있었다. `elementary` / `middle` 같은 값이 실제로 들어있었다. 그런데 애플리케이션 쪽 dataclass에는 이 필드가 없었다.

```python
@dataclass
class Unit:
    unit_id: int
    grade: int
    semester: int
    chapter: int
    title: str
```

SELECT 결과를 이 dataclass로 감싸다 보니 자연히 컬럼 목록에서도 빠졌고, WHERE 절에서도 빠졌다. 스키마는 알고 있었는데, 그 정보가 애플리케이션 레이어로 올라오는 도중에 조용히 사라진 것이다.

## 왜 이 조합이 유일하지 않은가

`(grade, semester, chapter)` 는 하나의 커리큘럼 안에서는 충분히 단원을 특정한다. 초등만 다루면 안 겹치고, 중등만 다뤄도 안 겹친다. 문제는 두 학교급을 **같은 테이블에 얹는 순간** 시작된다. `grade=1` 은 초1도 되고 중1도 된다. semester도, chapter 번호도 겹친다.

이건 사실 처음부터 있던 문제라기보다, 테이블이 자란 흔적에 가깝다. 처음엔 초등만 있었을 것이다. 그때는 `(grade, semester, chapter)` 로 충분했고, 이 세 값을 자연 키처럼 다루는 코드가 여기저기 생겼다. 그러다 중등이 들어오면서 스키마에는 `school_level` 이 추가됐다. 조회 코드는 그대로였다.

이런 종류의 확장에서 잘못되는 지점이 늘 비슷하다. 스키마는 새 축을 얻는데, 애플리케이션 레이어의 dataclass·역인덱스·조회 시그니처는 옛 축만 알고 있다.

## 고친 것

두 군데를 고쳤다. 먼저 dataclass에 `school_level` 을 되살리고, 조회 시그니처에도 넣었다.

```python
@dataclass
class Unit:
    unit_id: int
    school_level: str
    grade: int
    semester: int
    chapter: int
    title: str


def find_unit(
    school_level: str,
    grade: int,
    semester: int,
    chapter: int,
) -> Unit | None:
    return db.query(
        "SELECT * FROM units "
        "WHERE school_level=? AND grade=? AND semester=? AND chapter=?",
        (school_level, grade, semester, chapter),
    ).fetchone()
```

그다음으로, 자연어 파서가 뽑아낸 `("middle", 1, 2, 6)` 같은 튜플로 O(1) 조회를 하려고 인메모리 역인덱스도 같이 정비했다.

```python
unit_index: dict[tuple[str, int, int, int], Unit] = {}

for u in all_units:
    unit_index[(u.school_level, u.grade, u.semester, u.chapter)] = u
```

이제 "중1 2학기 6단원" 은 파싱 단계에서 `("middle", 1, 2, 6)` 이 되고, 결과는 항상 같은 단원이 나온다.

## 남은 것

같은 결의 코드가 다른 조회 경로에도 있는지 아직 다 훑지 못했다. 통계 추천이 아닌, 예를 들어 "이 단원과 이어지는 다음 단원" 같은 관계형 조회는 여전히 `(grade, semester, chapter)` 만 들고 다니는 자리가 있을 것 같다. 거기서는 아직 문제가 안 터졌을 뿐이지 같은 함정이다.

그리고 이 사건이 남긴 진짜 불편한 감각은 따로 있다. **스키마가 옳다고 해서 애플리케이션이 옳지는 않다**는 것. dataclass 하나에서 한 컬럼이 빠졌을 뿐인데, 그 아래로 이어지는 SELECT·WHERE·역인덱스·API 응답까지 전부 그 결손을 물려받았다. 축 하나를 추가하는 마이그레이션을 할 때, 스키마만 보지 말고 그 컬럼이 얼마나 많은 자리를 지나가야 하는지를 세는 습관을 붙여야겠다 — 아직 그 체크리스트는 못 만들었다.