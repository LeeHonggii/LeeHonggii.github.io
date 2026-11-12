---
title: "키워드 추출에서 끝내지 않기: 추천에 쓸 메타데이터를 만드는 과정"
date: 2026-11-12 17:00:00 +0900
categories: [AI]
tags: [recommendation, metadata, controlled-vocabulary, data-pipeline]
---

콘텐츠 추천 시스템의 기술 자료를 정리하다가 설명이 막힌 데가 있었다. "키워드는 어떻게 만들었나요?"라는 질문에 한 문장으로 답할 수 없었다. 텍스트에서 키워드를 뽑은 건 맞다. 하지만 추천 엔진이 읽는 값은 뽑아낸 키워드 그대로가 아니다.

이 글은 그 사이에 있는 단계를 풀어 쓴 것이다. 이번 작업은 엔진을 새로 만든 게 아니다. 이미 있는 구현을 읽으며 설명과 코드가 맞는지 확인하는 쪽이었다. 그래서 아래 코드는 엔진이 메타데이터를 읽어 들이는 지점이고, 어휘를 만드는 단계의 코드는 싣지 않는다. 그 단계는 말로만 설명한다.

## 추출은 첫 단계일 뿐이다

흐름을 한 줄로 쓰면 이렇다.

원본 → 풀 구축 → 콘텐츠 매칭 → 메타 4종 → 추천 엔진 입력

"키워드 추출"이라고 부르던 작업은 이 중 앞쪽 하나일 뿐이다. 나는 이걸 다섯 단계로 나눠서 보기로 했다.

1. **후보 어휘 추출**: 콘텐츠 텍스트에서 키워드 후보를 넉넉하게 뽑는다. 이 단계의 결과는 지저분하다. 같은 뜻의 표기가 여러 개고, 너무 구체적이거나 너무 일반적인 단어가 섞여 있다.
2. **의미 통합**: 같은 개념을 가리키는 후보를 하나로 묶는다.
3. **사람 검수**: 묶인 결과를 사람이 보고 남길 것과 버릴 것을 정한다.
4. **통제 어휘 확정**: 검수를 통과한 목록이 통제 어휘(controlled vocabulary, 미리 정해 둔 허용 목록)가 된다. 이후 태그는 이 목록 안에서만 나온다.
5. **콘텐츠 매칭**: 각 콘텐츠에 통제 어휘를 붙인다. 이번에는 텍스트에서 자유롭게 뽑지 않고 확정된 목록에서 고른다.

추출만 하고 멈추면 추천에는 쓸 수 없다. 자카드(집합 겹침) 같은 유사도는 두 콘텐츠가 **같은 표기**를 공유해야 점수가 나온다. "자료 분석"과 "데이터 분석"이 각자 다른 태그로 남아 있으면 엔진 입장에서는 겹치는 게 없다. 3~4단계를 건너뛰면 추천 품질 문제가 엔진 쪽 증상으로 나타나는데, 원인은 어휘 쪽에 있다.

## 메타 4종, 그리고 출처가 다른 속성

엔진이 읽는 메타데이터는 콘텐츠당 몇 가지 속성이다. 저장소의 로더가 쓰는 SQL을 보면 어떤 속성이 어디서 오는지 드러난다. 테이블명과 컬럼명은 일반화했다.

```python
_NCS_OK = "(CASE WHEN {c} IS NULL OR {c} = '' OR REPLACE({c}, '0', '') = '' THEN NULL ELSE {c} END)"
_NCS_EM = _NCS_OK.format(c='em.ncs_code')
_NCS_C = _NCS_OK.format(c='c.ncs_code')

_CONTENT_SQL = f"""
    SELECT c.contents_agent_pk AS cid,
           COALESCE(NULLIF(em.title, ''), c.title) AS title,
           COALESCE({_NCS_EM}, {_NCS_C}) AS ncs_code,
           c.tenant_id           AS institution,
           em.keyword_tags      AS keyword_tags,
           em.audience_tags     AS audience_tags,
           em.strand            AS strand,
           em.duration_bucket_4 AS duration_bucket_4,
           em.sub_category      AS sub_category,
           em.related_ncs       AS related_ncs,
           -- ...
           em.contents_agent_pk IS NOT NULL AS has_meta
    FROM content c
    LEFT JOIN content_meta em ON em.contents_agent_pk = c.contents_agent_pk
    WHERE c.tenant_id = %s
      AND c.contents_agent_pk IS NOT NULL
      AND c.title IS NOT NULL
      AND c.title <> ''
      AND c.title NOT LIKE '%%placeholder%%'
"""
```

`keyword_tags`, `audience_tags`, `strand`, `duration_bucket_4`는 메타 테이블에만 있는 컬럼이다. 원본 `content` 테이블에는 없으니 파이프라인이 새로 만든 값이다. 나는 이 네 개를 "메타 4종"으로 읽고 있다. 다만 같은 쿼리에 `sub_category`와 `related_ncs`도 있어서, 4종의 경계를 문서에 어떻게 그을지는 아직 확정하지 못했다.

반대로 분류 코드(`ncs_code`)와 제목은 원본에도 있고 메타에도 있다. 그래서 `COALESCE`로 메타 쪽을 먼저 보고, 비어 있으면 원본으로 떨어진다. 여기서 "비어 있다"의 기준이 따로 필요했다. `REPLACE(c, '0', '') = ''` 조건은 `000000` 같은 전부 0인 코드를 NULL로 취급한다. 값이 있는 것처럼 보이지만 실제로는 비어 있는 경우를 걸러낸다.

속성의 출처를 이렇게 나눠 두면 "이 값은 어디서 왔고 어떤 처리를 거쳤나"에 답하기 쉽다. 새로 생성한 속성은 위의 5단계를 거친 것이고, 원본에서 가져온 속성은 정규화만 거친 것이다.

## 정규화는 DB 밖에서 한 번 더

SQL에서 NULL 처리를 했다고 끝나지 않았다. 파이썬 쪽 변환 함수가 한 번 더 있다.

```python
def _as_list(v) -> list:
    if isinstance(v, str):
        try:
            return json.loads(v)
        except json.JSONDecodeError:
            return []
    return v or []

def _norm_ncs(v) -> str:
    v = (v or '').strip()
    if not v.isdigit():
        return ''
    if len(v) < 6:
        v = v.zfill(6)
    return '' if v == '000000' else v
```

`_as_list`는 JSON 컬럼이 문자열로 오든 이미 리스트로 오든 항상 리스트를 돌려준다. 깨진 JSON도 예외를 올리지 않고 빈 리스트로 처리한다. 태그가 없는 콘텐츠가 있는 건 정상이라서, 풀을 만드는 도중에 하나가 깨졌다고 전체 적재가 멈추면 안 된다고 판단했다. 대신 깨진 값이 조용히 "태그 없음"으로 바뀐다. 이게 문제가 될 수 있다. 아직 JSON이 깨진 행이 얼마나 되는지 세어 보지 않았다.

`_norm_ncs`는 숫자가 아니면 빈 문자열로 처리하고, 6자리보다 짧으면 앞을 0으로 채운다. 그리고 `000000`은 다시 빈 값으로 돌린다. 앞에 0을 채우는 이유는 코드를 숫자로 저장하는 과정에서 앞자리 0이 사라진 경우를 복구하기 위해서다. 그런데 복구 후에 전부 0이면 의미 없는 값이므로 다시 버린다. SQL과 파이썬이 같은 "빈 값" 규칙을 두 번 적용하는 셈이다. 중복이긴 하다. 하지만 SQL의 규칙은 두 컬럼 중 어느 쪽을 쓸지 고르는 용도이고, 파이썬의 규칙은 형식을 맞추는 용도라 일단 그대로 뒀다.

## 풀을 만들 때 세는 것

로더의 본체는 아래 함수다.

```python
def load_company_pool(tenant_id: str, conn=None) -> dict[str, dict]:
    own = conn is None
    conn = conn or get_connection()
    try:
        with conn.cursor() as cur:
            cur.execute(_CONTENT_SQL, (tenant_id,))
            rows = cur.fetchall()
    finally:
        if own:
            conn.close()

    pool: dict[str, dict] = {}
    labeled = 0
    for r in rows:
        pool[r['cid']] = _content_row(r)
        if r.get('keyword_tags'):
            labeled += 1

    logger.info('pool loaded: agent=%s size=%d (labeled=%d)', tenant_id, len(pool), labeled)
    return pool
```

이 함수에서 눈여겨본 건 로그 한 줄이다. `size`는 풀에 들어온 콘텐츠 수이고, `labeled`는 그중 키워드 태그가 하나라도 붙은 콘텐츠 수다. `LEFT JOIN`을 썼기 때문에 메타가 없는 콘텐츠도 풀에는 들어온다(`has_meta`로 구분). 추천 후보에서 빠지지는 않지만 태그 기반 점수는 받지 못한다.

이 두 숫자가 같은 개념이 아니라는 걸 문서에 명시하기로 했다. 메타데이터 작업을 설명할 때 자주 섞이는 숫자가 세 가지 있다.

- 통제 어휘에 들어 있는 **고유 키워드 수**
- 콘텐츠에 붙은 **태그 부착 횟수** (한 콘텐츠에 여러 개 붙으니 콘텐츠 수보다 훨씬 클 수 있다)
- 파이프라인이 **처리한 콘텐츠 수** (태그가 하나도 안 붙은 콘텐츠도 포함될 수 있다)

"키워드 몇 개를 만들었다"라고만 쓰면 이 중 무엇인지 알 수 없다. 면접에서 이 질문이 나오면 집계 단위부터 되묻게 될 것이다. 이번 글에는 세 숫자 모두 쓰지 않았다. 이번 작업 기록에 확인된 값이 없고, 확인되지 않은 값을 그럴듯하게 적을 수는 없기 때문이다.

## 남은 것

- 풀 로딩 로그의 `labeled`는 "키워드 태그가 비어 있지 않음"만 본다. 통제 어휘 밖의 값이 섞여 있는지는 이 지점에서 검증하지 않는다. 어휘 확정 단계에서 막았다고 믿고 있을 뿐이다. 엔진 입력 직전에 어휘 목록과 대조하는 검증을 넣을지 아직 정하지 못했다.
- 메타 4종의 경계(`sub_category`, `related_ncs`를 포함할지)는 문서에서 정리해야 한다.
- 세 가지 집계 숫자를 실제 데이터로 뽑아서 문서에 단위와 함께 적어 두는 일이 남았다.