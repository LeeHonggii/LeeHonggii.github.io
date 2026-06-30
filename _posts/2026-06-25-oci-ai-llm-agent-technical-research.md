---
title: "OCI로 교육 LLM Agent를 만든다면 먼저 확인해야 할 것들"
date: 2026-06-25 17:00:00 +0900
categories: [AI]
tags: [OCI, LLM, RAG]
---

# 도입 — 왜 이 글을 쓰는가

OCI(Oracle Cloud Infrastructure) 기반 교육 특화 LLM Agent 과제 수행계획서를 준비하면서, 막연히 "클라우드에서 LLM 쓰면 되지 않나?" 하고 시작했다가 꽤 많은 것을 조사해야 했다. 서비스 이름, 리전 가용성, 파인튜닝 방식, 데이터 레지던시 규정까지—제안서에 한 줄 쓰려면 열 줄을 확인해야 했다. 이 글은 그 조사 노트를 정리한 것이다.

---

## OCI AI 서비스 지형 파악하기

OCI에서 LLM Agent를 구성할 때 관련되는 주요 서비스는 다음과 같다.

| 서비스 | 역할 |
|---|---|
| **OCI Generative AI** | Cohere, Meta Llama 등 관리형 LLM 엔드포인트 |
| **AI Quick Actions** | 오픈소스 모델을 OCI Data Science에서 원클릭 배포 |
| **OCI OpenSearch** | 문서 검색 + 한국어 형태소 분석기(Nori) 지원 |
| **Oracle DB 23ai Vector Search** | 벡터 + 전통 SQL을 동일 DB에서 처리 |
| **OCI GenAI Agents** | RAG(검색 증강 생성) 파이프라인을 관리형으로 제공 |

처음에는 "Generative AI 하나만 쓰면 되겠지"라고 생각했는데, 교육 도메인은 한국어 문서 검색 정확도가 핵심이라 OpenSearch의 한국어 분석기 설정과 Vector Search를 어떻게 조합할지가 실제 설계 핵심이었다.

---

## 파인튜닝: 관리형 vs 자체 서빙

교육 특화 모델을 만들려면 도메인 데이터로 파인튜닝이 필요하다. OCI에서 선택지는 크게 두 가지다.

**관리형 파인튜닝 (OCI Generative AI T-Few / LoRA)**

OCI Generative AI 콘솔에서 학습 데이터를 업로드하면 LoRA(Low-Rank Adaptation, 모델 전체를 학습하지 않고 소수의 행렬만 조정하는 경량 파인튜닝 기법) 방식으로 커스텀 모델을 만들 수 있다. 학습 데이터 포맷은 JSONL이다.

```jsonl
{"prompt": "다음 학생 답안을 평가하라: ...", "completion": "채점 기준에 따라 3점. 근거: ..."}
{"prompt": "개념 설명 오류를 찾아라: ...", "completion": "2번 문장에서 ..."}
```

장점은 인프라 관리 부담이 없다는 것. 단점은 학습 데이터가 OCI 관리형 서비스 영역으로 이동한다는 것이다.

**오픈모델 자체 서빙 (AI Quick Actions)**

Llama 3나 Mistral 같은 오픈소스 모델을 AI Quick Actions로 OCI Data Science 환경에 배포하면, 데이터가 자사 테넌시 밖으로 나가지 않는다. 한국어 성능이 상대적으로 약한 모델도 있어 별도 검증이 필요하다.

| 항목 | 관리형 파인튜닝 | 자체 서빙 |
|---|---|---|
| 데이터 레지던시 | OCI 관리 영역 | 자사 테넌시 내 |
| 한국어 성능 | Cohere 기준 양호 | 모델마다 다름 |
| 운영 복잡도 | 낮음 | GPU 인스턴스 관리 필요 |

---

## Oracle DB 23ai Vector Search와 하이브리드 검색

RAG(Retrieval-Augmented Generation, 외부 문서를 검색해 LLM 응답 품질을 높이는 기법) 파이프라인에서 검색 정확도는 전체 시스템 품질을 좌우한다. Oracle DB 23ai는 기존 관계형 데이터와 벡터 검색을 같은 DB 안에서 처리할 수 있어, 교육 플랫폼처럼 구조화된 메타데이터(학년, 과목, 문항ID)와 비정형 텍스트를 함께 다루는 경우에 이점이 있다.

벡터 인덱스 생성 예시:

```sql
-- HNSW: 근사 최근접 이웃 탐색 알고리즘, COSINE: 벡터 유사도 측정 방식
CREATE VECTOR INDEX idx_doc_vec ON documents(embedding)
ORGANIZATION INMEMORY NEIGHBOR GRAPH
DISTANCE COSINE
WITH TARGET ACCURACY 95;
```

텍스트 키워드 검색과 벡터 의미 검색을 동시에 사용하는 하이브리드 검색은 `DBMS_HYBRID_VECTOR` 패키지로 구현한다. 교육 문서처럼 전문 용어가 많은 도메인에서는 키워드 검색과 의미 검색을 섞을 때 재현율이 눈에 띄게 올라갔다.

---

## 제안서 리스크: 기술보다 규정이 먼저다

기술 설계를 다 잡고 나서야 더 중요한 문제들이 보였다.

**한국 리전 가용성**: OCI GenAI 서비스가 한국 리전(ap-chuncheon-1)에서 전부 지원되지 않는다. 특정 기능은 미국 또는 독일 리전에서만 가용하다. 교육 데이터가 국외 리전으로 이동하면 개인정보보호법상 **국외 처리** 문제가 생긴다.

**비식별화**: 학생 답안, 성적 데이터는 LLM 학습 전에 반드시 비식별화가 필요하다. 이름, 학번 같은 직접 식별자뿐 아니라 준식별자(학교+학년+반 조합 등)도 검토해야 한다.

**감사 추적**: 공공·교육 기관 과제라면 AI가 어떤 근거로 어떤 답변을 생성했는지 로그가 남아야 한다. 재현 가능성을 위해 LLM 호출 시 `temperature=0`과 `seed` 고정을 기본값으로 설정하는 것이 좋다.

```python
response = llm.generate(
    prompt=prompt,
    temperature=0,   # 결정론적 출력
    seed=42          # 동일 입력 → 동일 출력 보장
)
```

---

## 마치며

OCI로 교육 LLM Agent를 제안하려면, 서비스 목록을 나열하는 것만으로는 부족하다. 어떤 데이터가 어느 리전으로 가는지, 파인튜닝 방식이 데이터 주권에 어떤 영향을 미치는지, 감사 요건을 어떻게 충족할지—이 질문들에 먼저 답할 수 있어야 기술 설계가 설득력을 갖는다. 기술 스택을 고르는 것과 과제를 통과할 수 있는 제안서를 쓰는 것은 다른 일이었다.