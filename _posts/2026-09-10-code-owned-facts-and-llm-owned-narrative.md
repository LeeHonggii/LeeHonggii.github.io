---
title: "LLM 리포트가 흔들리지 않게 만드는 법: 숫자는 코드가, 설명은 모델이"
date: 2026-09-10 17:00:00 +0900
categories: [AI]
tags: [LLM, 학습리포트, 프롬프트설계, Python, TypeScript]
---

LLM이 생성한 리포트를 처음 실무에 붙여봤을 때 제일 먼저 드는 의심은 이거다. "이 숫자, 다음에 다시 돌리면 같은 값이 나올까?"

틀린 의심이 아니었다. 같은 입력에 "3회 시도" 대신 "여러 번 시도"가 나오고, 직전에 "정답에 근접했습니다"라고 했던 문장이 다음 실행엔 사라져 있었다. LLM은 기본적으로 확률적 생성이고, 리포트에서 그 특성이 가장 위험하게 드러나는 것은 수치와 판정이다.

결국 물어야 할 질문은 하나다. **이 값은 누가 소유하는가.**

---

정답 판정 여부, 시도 횟수, 조작 이력 같은 값은 코드가 계산해야 한다. 틀렸냐 맞았냐는 채점 로직의 출력이지 모델의 해석이 아니다. 반면 "이 학생은 분모를 먼저 바꾸는 전략을 선택했다"는 문장이나 루브릭 서술은 LLM이 훨씬 잘한다. 룰로 짜면 경우의 수가 폭발하고, 짜도 어색하다.

이 경계를 명확하게 그으면 파이프라인이 생긴다.

```
케이스 로드
  → promptPiece 조립 (코드가 계산한 값들을 컨텍스트로 주입)
  → assess (LLM 호출 — 루브릭 평가, 서술 생성)
  → applyCodeFacts (코드 확정값으로 LLM 출력을 덮어씀)
  → render-both (교사용·학생용 두 버전 렌더링)
  → 저장
```

`applyCodeFacts`가 핵심이다. LLM이 뭔가 잘못 추론했더라도, 정답 여부·조작 횟수·점수 같은 필드는 코드가 마지막에 다시 써버린다. 모델의 자유도를 서술과 해석에만 제한하는 것이다.

---

추천 엔진 쪽 코드를 보면 이 패턴이 이미 정착돼 있다. 콘텐츠 깊이 점수를 계산하는 함수를 보자.

```python
MASTERY_BREADTH_T = 0.6
MASTERY_DEPTH_T = 0.5

def content_depth(c: dict, stats: dict) -> float:
    kws = c.get('keyword_tags') or []
    aud = c.get('audience_tags') or []
    a = max((AUD_STAGE.get(t, 0.0) for t in aud), default=0.0)
    sig = series_sig(c.get('title') or '')
    num = series_num(c.get('title') or '') or 0
    if not sig:
        s = 0.3
    elif num <= 2:
        s = 0.2
    else:
        tot = stats['series_total'].get(sig, num) or num
        s = min(num / tot, 1.0) if tot else 0.2
    r = statistics.mean([stats['rarity'].get(k, 0.0) for k in kws]) if kws else 0.0
    return round((a + s + r) / 3.0, 4)
```

대상 수준 신호(`a`), 시리즈 내 위치(`s`), 키워드 희소도(`r`) 세 값을 평균 낸다. 재실행해도 결과가 달라지지 않는다. 데이터가 안 바뀌면 점수가 안 바뀐다.

숙련도 레이블도 마찬가지다.

```python
def _mastery(breadth: float, depth: float) -> str:
    if breadth >= MASTERY_BREADTH_T and depth >= MASTERY_DEPTH_T:
        return '숙련'
    if depth >= MASTERY_DEPTH_T:
        return '넓힘 필요'
    if breadth >= MASTERY_BREADTH_T:
        return '깊이 보강'
    return '쌓는 중'
```

네 가지 레이블. 임계값 두 개. LLM에게 "이 사람 얼마나 숙련됐어요?"라고 물어보면 실행마다 다른 단어가 나온다. 임계값 기반 분기로 짜면 항상 같은 레이블이 나온다. 이게 코드가 소유해야 하는 판정의 전형이다.

---

풀 캐싱도 같은 맥락이다.

```python
@lru_cache(maxsize=config.POOL_CACHE_SIZE)
def _company_pool(tenant_id: str) -> dict:
    return load_company_pool(tenant_id)

@lru_cache(maxsize=config.POOL_CACHE_SIZE)
def _trend_raw(tenant_id: str) -> dict:
    pool = _company_pool(tenant_id)
    users: dict[str, set] = {}
    for uid, raw_cid, _ in _history_rows(tenant_id):
        if raw_cid in pool:
            users.setdefault(raw_cid, set()).add(uid)
    return {cid: len(us) for cid, us in users.items()}
```

트렌드 수치, 전이 확률, 콘텐츠 풀 — 이것들은 캐싱이 가능하다. 결정론적이기 때문이다. LLM의 "이 콘텐츠가 요즘 인기 있어 보입니다" 같은 추론이 아니라, `users.setdefault(raw_cid, set()).add(uid)` 로 센 숫자다.

---

리포트 파이프라인으로 돌아오면, `applyCodeFacts` 다음에 타입 검사가 들어간다.

```
npx tsc --noEmit && npm test
```

LLM이 반환한 JSON 구조가 타입에 맞는지, 코드 확정값이 제대로 덮였는지를 여기서 잡는다. LLM 출력을 신뢰하되, 검증은 코드가 한다는 원칙의 연장이다.

---

아직 정리 안 된 부분이 있다. 기초학력 쪽 활동 유형이 6개인데, 문항마다 주 유형 레이블이 아직 없다. 유형 없이 LLM에게 루브릭 평가를 맡기면 모델이 유형을 추론하게 되고, 그러면 판정 기준이 실행마다 흔들린다. 유형 레이블을 먼저 코드로 확정해야 `applyCodeFacts`가 의미가 있다.

그 레이블 작업이 지금 올라와 있다.