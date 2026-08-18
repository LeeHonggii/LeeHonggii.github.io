---
title: "중1 2학기 6단원이 초1 단원으로 잡힌 이유"
date: 2026-07-27 17:00:00 +0900
categories: [Backend]
tags: [식별자설계, 복합키, 도메인모델링]
---

추천 결과를 훑다가 눈이 걸렸다. 중1 2학기 6단원을 요청한 사용자에게, 로그상 조회된 단원명이 초등학교 1학년 것이었다.

처음엔 파싱 문제를 의심했다. `grade`, `semester`, `unit_order` 세 값이 어디선가 어긋났겠거니. 그런데 요청부터 저장소 계층 진입 직전까지 로그를 다 찍어봐도 `1, 2, 6` 은 정확히 그 값으로 흘러가고 있었다. 값이 맞는데 다른 행이 돌아온다. 그럼 답은 하나였다 — 그 세 값만으로는 애초에 하나의 행을 특정할 수 없었던 거다.

## "1학년"이 세 개인 나라

한국 교육과정에서 "1학년"은 초·중·고에 각각 존재한다. 여기까진 누구나 안다. 그런데 이걸 데이터 모델로 옮기면 이야기가 달라진다. `(grade, semester, unit_order)` 만으로 단원을 찾겠다고 결심한 순간, 세 학교급이 한 슬롯을 두고 겹친다.

| school_level | grade | semester | unit_order | 단원명 |
|---|---|---|---|---|
| elementary | 1 | 2 | 6 | 덧셈과 뺄셈 |
| middle | 1 | 2 | 6 | 일차방정식 |

두 행은 `school_level` 없이는 절대 구분되지 않는다. 스키마에는 이 컬럼이 있었다. `NOT NULL` 로 잘 박혀 있었다. 문제는 그 다음이었다.

## dataclass, SELECT, WHERE — 세 곳의 침묵

저장소 계층을 열었다. 단원을 담는 dataclass에 `school_level` 이 없었다.

```python
@dataclass
class CurriculumUnit:
    grade: int
    semester: int
    unit_order: int
    unit_name: str
```

이 dataclass를 채우는 SELECT 도 마찬가지였다.

```sql
SELECT unit_name, grade, semester, unit_order
FROM curriculum_units
WHERE grade = ? AND semester = ? AND unit_order = ?
```

`ORDER BY` 도 없다. `LIMIT` 도 없다. DB가 먼저 뱉는 행이 정답이 된다. 우연히 초등 데이터가 먼저 적재됐고, 그래서 `1,2,6` 은 항상 초등 덧셈으로 귀결됐다. 중학교 사용자에게 초등 단원이 붙는 이 조용한 사고는, 인덱스가 결정론적으로 잘못된 방향을 가리키는 동안 아무 예외도 던지지 않았다.

이게 이 버그의 진짜 얼굴이다. 실패가 아니라 **성공을 가장한 오답**.

## 고친 자리

수정은 한 줄이 아니었다. 도메인 식별자가 하나 부족하다는 걸 인정하면, 그 결정이 스민 자리를 다 손봐야 한다.

```python
@dataclass
class CurriculumUnit:
    school_level: str   # "elementary" | "middle" | "high"
    grade: int
    semester: int
    unit_order: int
    unit_name: str
```

```sql
SELECT unit_name, school_level, grade, semester, unit_order
FROM curriculum_units
WHERE school_level = ?
  AND grade = ?
  AND semester = ?
  AND unit_order = ?
```

호출부 시그니처에도 `school_level` 을 강제 인자로 넣었다. 파이썬 타입 체커가 누락된 호출을 곧바로 지적하게 하려는 의도였다. 런타임에서 조용히 틀리는 것보다, 정적 단계에서 시끄럽게 틀리는 편이 훨씬 낫다.

회귀 테스트는 딱 한 문장을 검증한다 — "초등 1-2-6 과 중등 1-2-6 이 각각 자기 단원으로 돌아온다." 이전 테스트 스위트에는 이 케이스가 없었다. 그러니 버그가 살아남을 수 있었다.

## 남는 물음

이 사건 이후 스키마 리뷰 때 생긴 습관이 있다. 자연키 후보를 볼 때 "이 값들의 조합이 도메인 전체에서 유일한가?" 를 큰 소리로 물어보게 됐다. 여기서 "도메인 전체"란 지금 있는 데이터가 아니라, **이 테이블이 커버해야 하는 세계의 범위**다.

아직 정리 못 한 게 있다. 교육과정 개정판이 들어오면 `(school_level, grade, semester, unit_order)` 조합조차 시간축을 만나 다시 겹친다. 2015 개정 초1 2-6 과 2022 개정 초1 2-6 은 다른 단원일 수 있다. `curriculum_version` 을 복합키에 넣을지, 아니면 단원 테이블 자체를 버전별로 분리할지 — 여기까지는 아직 손을 못 댔다. 데이터가 들어오는 걸 보고 결정할 참이다.