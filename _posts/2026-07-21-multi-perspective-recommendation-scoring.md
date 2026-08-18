---
title: "추천 축을 나누면 무엇이 달라질까: 직업·관심·선호 점수 설계 해부"
date: 2026-07-21 17:00:00 +0900
categories: [AI]
tags: [recommendation, ranking, ncs, python]
---

## 하나의 점수로 세 가지 질문에 답하려 했다

처음엔 유사도 하나로 다 될 줄 알았다. 사용자와 콘텐츠를 벡터로 만들고, 코사인 찍고, 정렬해서 위에서 자르면 되는 거 아닌가. 실제로 그렇게 짜서 돌려봤다.

돌아온 결과가 이상했다. 어떤 사람에겐 자기가 이미 본 것과 거의 똑같은 콘텐츠가 올라왔고, 어떤 사람에겐 자기 분야와 상관없는 게 섞여 들어왔다. 점수가 약한 게 아니었다. 애초에 "무엇을 추천할 것인가"가 사람마다 다른 질문이었는데, 나는 그걸 한 공식으로 뭉개고 있었다.

- 지금 하는 일과 같은 결의 콘텐츠가 더 보고 싶다 — **깊이**
- 요즘 관심 생긴 분야가 있는데 뭐가 있나 — **인접 탐색**
- 내가 평소에 좋아하는 스타일 — **소비 패턴**

이 세 개는 서로 다른 대역을 봐야 한다. 그래서 축을 세 개로 쪼갰다. 직업·관심·선호. 각 축이 후보를 뽑는 방식도, 가중치도 따로 간다.

## 후보 풀부터가 축마다 다르다

축을 나눈다는 건 산식만 나누는 게 아니라 **애초에 어떤 콘텐츠 집합을 대상으로 점수를 매길지**를 나눈다는 뜻이다. 그래서 풀 로딩 자체를 축별로 필터링 가능한 구조로 만들었다.

```python
@lru_cache(maxsize=config.POOL_CACHE_SIZE)
def _company_pool(tenant_id: str) -> dict:
    return load_company_pool(tenant_id)


@lru_cache(maxsize=config.POOL_CACHE_SIZE)
def _inst_pool(tenant_id: str) -> dict:
    return {cid: c for cid, c in _company_pool(tenant_id).items()
            if c.get('content_type') == 'INSTITUTION' and c.get('has_meta')}
```

전체 풀은 테넌트 단위로 한 번 읽어 캐시한다. 그리고 그 위에 **메타데이터가 붙어 있는 것만** 걸러낸 서브풀을 따로 만든다. 왜 이렇게 이중으로 잡아두냐면, 관심·선호 축은 태그·스트랜드 같은 라벨이 없으면 점수를 계산할 방법이 없기 때문이다. 라벨 없는 콘텐츠는 아예 후보에서 빼는 게 맞다.

반면 직업 축은 NCS 코드만 있어도 최소한의 사다리 유사도를 매길 수 있어서 좀 더 넓은 풀을 본다.

`has_meta` 를 계산하는 쪽은 SQL 안에서 처리했다.

```sql
SELECT ...,
       em.source_system IS NOT NULL AS has_meta
FROM content ec
LEFT JOIN content_meta em ON em.source_system = ec.content_uuid
LEFT JOIN content_detail cd ON cd.source_system = ec.content_uuid
WHERE ec.tenant_id = %s
  AND ec.contents_name IS NOT NULL
  AND ec.contents_name <> ''
  AND ec.contents_name NOT LIKE '%%placeholder%%'
```

`LEFT JOIN` 으로 붙여놓고 존재 여부만 flag로 뽑는다. Python 쪽에서 두 번 조회하지 않고, 한 번의 pool 로딩으로 축별 서브풀을 파생시킬 수 있게 만든 게 캐시 히트율에 크게 영향을 준다. 테넌트마다 풀을 다시 읽는 건 초기 몇 초를 잡아먹는다.

## 정규화가 절반이다

축별 점수를 짜기 전에 훨씬 손이 많이 간 게 정규화 단계였다. 원본 데이터가 지저분해서, 이걸 그대로 넣으면 자카드든 사다리든 다 오작동한다.

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

`keyword_tags` 는 JSON 문자열로 들어올 때도 있고 이미 리스트인 경우도 있다. NCS 코드는 5자리로 들어오는 것, 앞자리 0이 잘린 것, `'000000'` 같은 사실상 null 값이 섞여 있다. `'000000'` 을 그대로 두면 이 코드를 가진 콘텐츠들이 서로 사다리에서 완벽 일치로 튀어나온다. 초기에 이걸 못 보고 결과 앞쪽에 미분류 콘텐츠가 몰려나온 적이 있다.

이런 정규화가 뒤에 나오는 모든 축 점수의 전제조건이 된다. 지저분한 데이터 위에 정교한 산식을 얹어봐야 소용없다는 걸, 자카드 유사도를 튜닝하다가 알았다.

## 직업 축 — 겹치는 걸 강화한다

직업 축의 목적은 명확하다. 지금 이 사람의 직무 방향에서 **더 깊게** 파고들 콘텐츠를 찾는다.

```
S_H(job) = 0.5·키워드자카드 + 0.3·NCS사다리 + 0.2·스트랜드일치
```

- 키워드 자카드는 사용자 프로필의 태그 집합과 콘텐츠 `keyword_tags` 의 교집합/합집합
- NCS 사다리는 국가직무능력표준 분류 코드의 계층 거리 (상위 대분류 일치 → 중분류 일치 → 소분류 일치로 계단식 가중)
- 스트랜드 일치는 학습 단위 묶음이 같으면 boolean 가산

가중치 절반을 키워드에 준 건, 실제 라벨링된 태그가 직무 방향성에 가장 직접적인 신호였기 때문이다. NCS는 분류가 넓어서 같은 소분류 안에서도 결이 꽤 다르다. 사다리를 주 신호로 쓰면 "같은 직무군인데 방향이 어긋난 콘텐츠"가 상위에 걸린다.

## 관심 축 — 겹치는 걸 일부러 뺀다

관심 축이 재미있는 지점이다. 직업 축과 **정반대 전략**을 쓴다. 사용자가 이미 소비했거나 프로필상 익숙한 키워드는 후보에서 감점하거나 아예 제외한다. 대신 인접한 미지의 영역을 뽑는다.

이걸 가능하게 하는 게 관계 매트릭스다.

```python
@lru_cache(maxsize=1)
def _relation() -> RelationMatrix:
    return RelationMatrix.load(Path(__file__).resolve().parent / 'relation_matrix.json')
```

사전 구축된 직무-역량 그래프를 파일에서 한 번 로드하고 프로세스 수명 내내 재사용한다. 현재 직무 코드에서 1~2홉 떨어진 인접 코드들을 후보 소스로 삼고, 이미 아는 키워드와 겹치는 콘텐츠는 뒤로 밀어낸다.

이 필터가 없을 때가 실제로 있었는데, 사용자가 "관심 분야 추천"을 눌러도 자기가 이미 본 것과 태그가 80% 겹치는 결과가 나왔다. 산식은 다르지만 후보 풀이 같으면 결국 비슷한 답이 나온다. **축을 나눈다는 건 산식보다 후보 소스를 다르게 가져가는 게 더 크다**는 걸 이 축에서 배웠다.

## 선호 축 — 라벨 대신 이력을 본다

선호 축은 프로필 라벨을 거의 안 본다. 대신 실제 소비 이력에서 패턴을 추출한다.

```python
_TRANSITION_WINDOW = 10
_PEER_MIN_SUPPORT = 5


@lru_cache(maxsize=config.POOL_CACHE_SIZE)
def _transition_raw(tenant_id: str) -> dict:
    pool = _company_pool(tenant_id)
    by_user: dict[str, list[str]] = {}
    for uid, raw_cid, _ in _history_rows(tenant_id):
        if raw_cid in pool:
            by_user.setdefault(uid, []).append(raw_cid)

    outgoing: dict[str, int] = {}
    pair_users: dict[tuple[str, str], set[str]] = {}
    for uid, seq in by_user.items():
        if len(seq) < 2:
            continue
        for i, src in enumerate(seq[:-1]):
            outgoing[src] = outgoing.get(src, 0) + 1
            for dst in seq[i + 1:i + 1 + _TRANSITION_WINDOW]:
                if dst == src:
                    continue
                pair_users.setdefault((src, dst), set()).add(uid)
```

사용자 시퀀스를 훑으면서 A → B 로 이어진 쌍을 모으고, `_PEER_MIN_SUPPORT = 5` 명 이상 겹친 쌍만 살린다. 창(window) 10은 세션 안에서 앞뒤로 이어질 만한 거리를 대충 잡은 값인데, 이건 재봐야 한다. 아직 A/B 로 창 크기를 비교해보진 않았다.

`support` 를 사람 수로 세는 게 중요하다. 처음엔 페어 발생 횟수(count)로 셌는데, 한 사람이 짧은 시간에 같은 경로를 반복 소비하면 이 쌍 하나가 전체 통계를 왜곡했다. **유저 집합의 크기**로 바꾸면서 그 문제가 사라졌다.

여기서 뽑힌 전이 강도가 선호 축의 이웃 후보를 만든다. 명시적 태그와 무관하게, "이 콘텐츠를 본 사람들이 다음으로 본 것"에 가까운 결과가 올라온다.

## 축을 합칠 때 — 그리고 아직 안 정한 것

세 축이 각각 후보와 점수를 뽑으면 마지막에 합친다.

```
축 점수 = w_H·S_H + w_P·S_P + w_T·S_T
최종 점수 = 축 점수 × (1 + Σ부스팅)
```

부스팅은 최신성·인기도 같은 부가 신호를 곱셈으로 얹는다. 덧셈이 아니라 곱셈으로 얹은 이유는, 축 점수가 0에 가까우면 아무리 인기 있어도 올라오지 말아야 하기 때문이다. 관련 없는 인기 콘텐츠가 상위에 뜨는 걸 막는 안전장치다.

가중치 `w_H`, `w_P`, `w_T` 는 아직 정하지 못했다. 지금은 사용자 요청 컨텍스트에 따라 스위칭한다 — "관심 분야 추천"이면 `w_P` 를 크게, 기본 홈이면 세 축을 균등에 가깝게. 이 부분을 사용자 세그먼트별로 학습해서 자동 조정하는 게 다음 할 일인데, 그러려면 세그먼트 단위 CTR 로그가 안정적으로 쌓여야 한다. 그건 아직 재보지 않았다.

한 가지 남은 애매함은, 축 간 후보가 겹칠 때 어떻게 처리할지다. 지금은 최종 랭킹 단계에서 dedupe 만 하는데, 이러면 "직업 축에서도 상위, 선호 축에서도 상위"인 콘텐츠가 각 축 관점에서 봤을 땐 자연스럽지만 사용자 눈엔 한 번밖에 안 보인다. 축별 슬롯을 나눠 노출할지, 그냥 합쳐서 순위 매길지는 지표를 더 봐야 답이 나올 것 같다.