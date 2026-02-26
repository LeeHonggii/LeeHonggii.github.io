---

title: "LLM 기반 콘텐츠 검색 시스템: 카테고리 분류 vs 메타데이터 직접 매칭"
date: 2025-02-26 09:30:00 +0900
categories: [Daily, LLM]
tags: [search, metadata, gpt, gemini, claude, template-matching, prompt-engineering]
---

## 안녕하세요

교육용 콘텐츠 검색 시스템을 만들면서 느낀 건 하나였습니다.

> 카테고리를 잘 나누는 게 아니라,
> 검색 구조를 어떻게 설계하느냐가 진짜 문제다.

처음엔 "LLM으로 카테고리 분류하면 되겠지"라고 생각했습니다.
Gemini, Claude, GPT 세 모델로 각각 카테고리 기준을 만들어보고,
질문을 던지면 알아서 분류해주는 구조를 상상했습니다.

그런데 실제로 돌려보니 전혀 다른 그림이 나왔습니다.

* 카테고리 매칭만으로는 정확도가 부족
* 같은 카테고리 내에서도 의도와 맞지 않는 결과
* 검증 단계를 추가하면 복잡도만 증가

이번 글은 이걸 **실제 테스트 + 코드 중심으로** 정리한 기록입니다.

---

# 1. 테스트 개요

약 100개의 교육용 템플릿을 대상으로 검색 시스템을 테스트했습니다.

| 방식 | 설명 |
| -- | -- |
| 방식 1 | 모델별 카테고리 분류 (Gemini/Claude/GPT) |
| 방식 2 | 메타데이터 기반 직접 매칭 |

목표는 단순했습니다:

> 사용자가 "퍼즐로 도형 만들기"라고 검색하면
> 관련 템플릿 3~6개를 정확하게 찾아주는 것.

---

# 2. 방식 1: 모델별 카테고리 분류

세 가지 모델에게 각각 카테고리 기준을 생성하게 했습니다.

### Gemini 분류 기준

* 수와 연산 (자릿값, 덧셈, 분수, 소수...)
* 도형 및 공간 (평면도형, 모양만들기)
* 변화와 관계 (규칙찾기, 집합)
* 측정 (시계, 길이)

### Claude 분류 기준

* 01_시계_시간
* 02_저울_비교
* 03_도형_퍼즐
* ... 총 10개 카테고리

### GPT 분류 기준

* 01_도형구성퍼즐
* 02_도형이동변환
* ... 총 12개 카테고리

같은 데이터인데 모델마다 분류 체계가 다릅니다.

---

# 3. 방식 1의 한계

실제로 돌려보니 문제가 있었습니다.

```
검색: "퍼즐로 삼각형 만들기"

결과: 퍼즐 카테고리 전체 6개 랜덤 출력
  → 칠교, 펜토미노, 패턴블록 섞여서 나옴
  → 사용자가 원한 건 칠교판인데...
```

문제점을 정리하면:

* 카테고리 매칭만으로는 정확도 부족
* 같은 카테고리 내 랜덤 출력 → 의도와 불일치
* 검증 단계 추가 시 → 2단계 구조로 복잡도 증가

```python
# 방식 1 구조
def search_v1(query):
    category = classify_category(query)  # 1단계
    candidates = get_templates_by_category(category)
    results = verify_relevance(query, candidates)  # 2단계
    return results
```

> 카테고리 선택 후 다시 검증하는 2단계 구조가 되면서
> 복잡도는 올라가고, 여전히 정확도는 불안정했습니다.

---

# 4. 방식 2: 메타데이터 기반 직접 매칭

발상을 바꿨습니다.

> 카테고리로 나누지 말고,
> 전체 메타데이터를 GPT에게 주고 직접 선택하게 하자.

### 메타데이터 구성

각 템플릿별로 다음 정보를 추출했습니다:

```python
@dataclass
class Template:
    project_id: str
    project_title: str
    tool_names: list      # 사용 도구: 칠교판, 시계, 저울...
    canvas_objects: list  # 캔버스 객체: 삼각형, 원, 선...
    activity_type: str    # 활동 유형: 도형 구성, 수 비교...
    learning_goal: str    # 학습 목표
    keywords: list        # 관련 키워드
```

### 메타데이터 생성 방법

두 가지 소스를 활용했습니다:

1. **DB에서 직접 추출**
   * 도구 정보: svgId 필드 파싱
   * 객체 정보: type/n 필드 파싱 (n=3 → 삼각형)

2. **LLM 이미지 분석**
   * 썸네일 이미지를 Gemini에 전달
   * 활동 유형, 학습 목표, 키워드 자동 생성

```python
# 객체 추출 예시
def extract_objects(contents_json):
    objects = set()
    for item in contents_json:
        if item.get("type") == "circle":
            objects.add("원")
        elif item.get("n") == 3:
            objects.add("삼각형")
        elif item.get("n") == 4:
            objects.add("사각형")
    return list(objects)
```

---

# 5. 검색 구조 비교

### 방식 1 (카테고리)

```
질문 → 카테고리 분류 → 해당 카테고리 템플릿 → 검증 → 결과
        (1단계)                              (2단계)
```

### 방식 2 (메타데이터)

```
질문 + 전체 메타데이터 → GPT 직접 선택 → 결과
              (1단계)
```

방식 2가 훨씬 단순합니다.

```python
# 방식 2 구조
def search_v2(query, all_templates):
    prompt = build_prompt(query, all_templates)
    response = gpt_call(prompt)
    return parse_results(response)
```

---

# 6. GPT 프롬프트 설계

핵심은 GPT가 **직접 선택**하게 만드는 것입니다.

```python
system_prompt = f"""
당신은 교육용 템플릿 추천 도우미입니다.

## 템플릿 목록 ({len(templates)}개)

```json
{metadata_json}
```

## 응답 규칙
1. 사용자 요청에 맞는 템플릿 3~6개 선택
2. 관련도 높은 순서로 정렬
3. JSON 형식으로만 응답

## 응답 형식
{{
  "selected_templates": [
    {{
      "idx": 0,
      "reason": "선택 이유",
      "relevance": 0.95
    }}
  ],
  "summary": "추천 결과 요약"
}}
"""
```

중요한 점:

* `response_format={"type": "json_object"}` 사용
* idx는 메타데이터 인덱스와 매칭
* relevance로 관련도 수치화

---

# 7. 테스트 결과

### 검색 예시 1

```
질문: "퍼즐로 삼각형 만들기"

방식 1 결과: 퍼즐 카테고리 6개 랜덤
방식 2 결과:
  1. 칠교판_기본 (관련도 95%)
  2. 칠교판_응용 (관련도 90%)
  3. 패턴블록_삼각형 (관련도 85%)
```

### 검색 예시 2

```
질문: "시계 읽기 연습"

방식 1 결과: 측정 카테고리 전체
방식 2 결과:
  1. 시계_정각 (관련도 95%)
  2. 시계_30분 (관련도 90%)
  3. 시계_5분단위 (관련도 85%)
```

### 비교 정리

| 항목 | 방식 1 (카테고리) | 방식 2 (메타데이터) |
| -- | -- | -- |
| 검색 단계 | 2단계 | 1단계 |
| 정확도 | 중간 | 높음 |
| 복잡도 | 높음 | 낮음 |
| GPT 토큰 | 적음 | 많음 |
| 확장성 | 카테고리 추가 | 메타데이터 추가 |

> 토큰은 더 쓰지만,
> 단순하고 정확한 방식 2가 실용적이었습니다.

---

# 8. 확장 방안

현재 100개 수준에서는 메타데이터 직접 매칭이 충분합니다.

500개 이상으로 늘어나면:

### 방안 1: 의도 분류기 추가

```python
def search_v3(query, templates):
    intent = classify_intent(query)  # 도구/활동/주제
    candidates = filter_by_intent(templates, intent)
    return gpt_select(query, candidates)
```

### 방안 2: 벡터 임베딩 검색

```python
def search_v4(query, templates):
    query_embedding = embed(query)
    candidates = vector_search(query_embedding, top_k=20)
    return gpt_select(query, candidates)
```

### 방안 3: 2단계 필터링

```python
def search_v5(query, templates):
    # 1단계: 키워드 필터
    candidates = keyword_filter(query, templates)
    # 2단계: GPT 선택
    return gpt_select(query, candidates)
```

---

# 정리

이번 작업에서 확실해진 건 이겁니다.

1. **카테고리 분류는 생각보다 부정확하다.**
   * 모델마다 기준이 다르고
   * 같은 카테고리 내에서도 의도 불일치 발생

2. **메타데이터 직접 매칭이 더 단순하고 정확하다.**
   * 1단계로 끝남
   * GPT가 전체를 보고 직접 판단

3. **메타데이터 품질이 핵심이다.**
   * DB 필드 파싱 + LLM 이미지 분석
   * 도구 없는 템플릿도 객체 정보로 커버

4. **확장은 나중에 고민해도 된다.**
   * 100개 수준 → 직접 매칭 충분
   * 500개 이상 → 의도 분류기 또는 벡터 검색

이제는 카테고리를 더 잘 나누는 게 아니라,
메타데이터를 더 풍부하게 만드는 방향으로 넘어가고 있습니다.
