---
title: "추천할 때마다 전체 데이터를 읽어야 할까? 메타데이터 사전 집계로 바꾼 설계"
date: 2026-09-30 17:00:00 +0900
categories: [Backend]
tags: [recommendation, precomputed-aggregation, metadata, python]
---

추천 요청이 들어올 때마다 원본 콘텐츠 테이블을 훑는 구조를 다시 보기로 했다. 추천 대상은 수십 개 단위의 공급자이고, 콘텐츠는 그 아래 수만 건 단위로 쌓여 있다. 사용자가 질문지에 답하면 "이 사람에게 맞는 공급자"를 고르는데, 그 판단에 필요한 건 콘텐츠 한 건 한 건이 아니다. 공급자마다 어떤 분야를 얼마나 다루는지, 강의 길이는 어떤지 같은 **분포**다. 그걸 요청마다 다시 세고 있었다.

그래서 요청 시점의 계산과 사전 집계의 역할을 나누기로 했다. 공급자별 메타데이터를 미리 만들어 JSON 한 파일로 떨어뜨리고, 추천은 그 파일만 읽는다. 이 글은 그 빌드 스크립트와, 질문지를 왜 그 메타데이터에 맞춰 짜게 됐는지에 대한 학습 노트다.

## 원본을 읽는 대신 무엇을 미리 세나

메타데이터는 두 종류 재료로 만든다. 하나는 콘텐츠 테이블에 이미 붙어 있는 분류값을 세는 것이다. 다른 하나는 공급자 소개문에서 LLM으로 뽑아 둔 학습 성향 점수다. 앞쪽은 전수 집계로 나오는 사실이고, 뒤쪽은 추론이다. 둘을 한 구조에 섞되 출처가 다르다는 건 기억해 둬야 했다.

먼저 원본에서 집계 대상 행을 가져오는 쿼리다.

```python
cur.execute("""
    select c.provider_id, m.strand, m.job_code, m.keyword_tags, m.duration_bucket_4
      from content c
      join content_meta m on m.content_key = c.content_id
     where c.change_state <> 'D' and c.is_placeholder = 0
       and m.strand is not null and m.strand <> ''""")
rows = cur.fetchall()
```

삭제 표시(`change_state <> 'D'`)된 행, 자리만 잡아 둔 placeholder 행, 대분류가 비어 있는 행을 여기서 걷어낸다. 집계는 "많이 다루는 쪽"을 드러내는 작업이라 쓰레기 행이 섞이면 분포가 그대로 오염된다. 걸러내는 위치를 집계 루프 안이 아니라 쿼리로 잡은 건 그래서다. 나중에 "왜 이 공급자 분포가 이상하죠"를 추적할 때 조건이 SQL 한 곳에만 있으면 편하다.

공급자 이름은 다른 스키마에 있어서 별도 연결로 한 번 더 읽는다. DB 간 조인은 하지 않고 파이썬에서 딕셔너리로 붙였다. 이름이 비어 있으면 ID를 그대로 쓴다.

## 정규화는 작은 함수 두 개로

원본 값이 깨끗하지 않았다. 키워드 태그는 JSON 문자열로 들어오기도 하고 이미 리스트이기도 하다. 직무 분류 코드는 앞자리 0이 잘려 있거나 전부 0인 자리표시 값이 섞여 있다.

```python
def as_list(v):
    if isinstance(v, str):
        try:
            v = json.loads(v)
        except json.JSONDecodeError:
            return []
    return [str(x).strip() for x in (v or []) if str(x).strip()]


def job_code6(v):
    x = str(v or "").strip()
    if not x.isdigit():
        return ""
    x = x.zfill(6)
    return "" if set(x) == {"0"} else x
```

`as_list`는 파싱에 실패하면 예외를 던지지 않고 빈 리스트를 돌려준다. 태그 한 건이 깨졌다고 전체 빌드가 멈추면 곤란하고, 태그가 빠지면 그 건이 카운트에 안 들어갈 뿐이다. 반면 `job_code6`는 숫자가 아니면 버리고, `zfill(6)` 후 전부 0이면 버린다. `000000`을 유효한 코드로 세면 "이 공급자는 분류 불명 콘텐츠가 많다"가 마치 하나의 직무 분야처럼 보인다. 이런 값은 조용히 틀린 채로 추천에 들어가는 종류라서 경계에서 끊었다.

## 공급자별 네 개의 분포

```python
for pk, v in by.items():
    if len(v) < a.min_content:
        continue
    S, N, K, D = (collections.Counter() for _ in range(4))
    for r in v:
        S[r["strand"]] += 1
        code = job_code6(r["job_code"])
        if code:
            N[code] += 1
        for w in as_list(r["keyword_tags"]):
            K[w] += 1
        if r["duration_bucket_4"]:
            D[str(r["duration_bucket_4"])] += 1
    lm = intro.get(pk, {}).get("learning_goal", {})
    out[pk] = {
        "provider_name": names.get(pk) or pk,
        "content_n": len(v),
        "strand_share": dict(S.most_common()),
        "job_share": dict(N.most_common()),
        "keyword_share": dict(K.most_common()),
        "duration_share": dict(D.most_common()),
        "learning_mode": {k: x["score"] for k, x in lm.items()},
    }
```

대분류, 직무 코드, 키워드, 강의 길이 구간을 각각 `Counter`로 센다. 값은 비율이 아니라 **건수**로 저장했다. `share`라는 이름을 붙여 놓고 건수를 넣은 건 솔직히 어색하다. 건수를 저장해 두면 `content_n`으로 나눠 비율을 언제든 만들 수 있고, 콘텐츠가 3건뿐인 공급자의 100%와 3천 건짜리의 100%를 구분할 수 있어서였다. 필드 이름은 바꾸는 게 맞을 것 같다.

`learning_mode`는 집계가 아니다. 소개문에서 LLM으로 추출해 별도 JSON에 저장해 둔 성향 점수를 그대로 옮겨 온다. 빌드 스크립트는 점수의 `score`만 꺼내 붙이고, 추출 자체는 여기서 하지 않는다. 호출과 추출은 앞 단계, 집계는 이 단계로 경계를 나눴다.

## 만든 뒤에 확인하는 것

```python
raw = json.dumps(out, ensure_ascii=False, separators=(",", ":")).encode()
print(f"공급자 {len(out)}곳 · 콘텐츠 {len(rows):,}건 · 전체 {len(raw)/1024:.0f}KB")
print(f"learning_mode 비어 있는 공급자 {[k for k, v in out.items() if not v['learning_mode']]}")
```

빌드가 끝나면 결과 크기를 KB로 찍고, 성향 점수가 비어 있는 공급자 목록을 찍는다. 크기를 찍는 이유는 이 파일을 통째로 메모리에 올려 쓸 생각이기 때문이다. 요청마다 전체 테이블을 읽는 것보다 싸야 의미가 있는 설계다. 다만 이번 글에는 실제 KB 수치나 건수를 적지 않는다. 출력 값을 이 글 작성 시점에 확인해 옮겨 두지 않았고, 요청 지연 시간을 전후로 비교하는 측정도 아직 하지 않았다. "빨라졌다"고 쓸 근거는 없다.

성향 점수 누락 목록을 따로 찍는 건, 소개문이 없거나 추출이 실패한 공급자가 조용히 "성향 없음"으로 추천에서 불리해지는 걸 막으려는 장치다. 지금은 찍어서 눈으로 보는 수준이다.

## 질문지는 메타데이터에서 거꾸로 나온다

집계 구조를 잡고 나니 질문지를 보는 시선이 달라졌다. 처음엔 사용자에게 물어보기 쉬운 것부터 나열하는 식이었다. 그런데 그 답을 메타데이터의 어느 필드와 비교할지 그려 보면 이어지지 않는 질문이 나왔다. 답은 받았는데 공급자 간 차이를 만들지 못하는 질문이다.

그래서 기준을 뒤집었다. 공급자들을 갈라놓는 축이 분야 분포, 직무 코드, 키워드, 강의 길이, 학습 성향이라면, 질문은 그 축 중 하나에 대응해야 한다. 대응하는 필드가 없는 질문은 빼거나 메타데이터 쪽에 필드를 추가한다. 반대로 어떤 질문으로도 묻지 않는 필드는 빌드에서 계산할 이유가 약해진다. 질문지와 메타데이터를 한 세트로 설계한다는 게 이 뜻이다.

## 아직 남은 것

- `strand_share` 같은 필드 이름이 건수를 담고 있다. 이름을 고칠지, 비율로 바꿀지 정해야 한다.
- 사전 집계는 원본이 바뀌면 낡는다. 지금은 스크립트를 다시 돌리는 방식이고, 갱신 주기는 정하지 않았다.
- 분포가 같아 보이는 공급자를 질문지가 실제로 갈라내는지는 데이터로 확인하지 못했다. 설계상 그래야 한다는 단계까지다.
- 키워드 상위 태그가 공급자마다 얼마나 겹치는지 아직 안 봤다. 많이 겹치면 키워드 축은 변별력이 약할 수 있다.