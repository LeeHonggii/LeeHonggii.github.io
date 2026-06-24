---
title: "OCI로 교육 LLM Agent 만들기 전에 반드시 확인해야 할 것들: 리전, 튜닝, RAG, 데이터 레지던시"
date: 2026-06-24 17:00:00 +0900
categories: [AI]
tags: [OCI, LLM-Agent, RAG, 파인튜닝, 데이터레지던시]
---

# 도입 (왜 이 글)

Oracle Cloud Infrastructure(OCI) 기반으로 교육용 LLM Agent를 구성해야 하는 상황이 생겼다. 처음엔 "클라우드 AI니까 API 연결하면 되겠지"라고 생각했다. 착각이었다.

실제로 조사해보니 서비스 가용 리전, 파인튜닝 경로의 차이, RAG 구성 요소, 데이터 레지던시(Data Residency, 데이터가 특정 국가·리전 밖으로 나가지 않도록 보장하는 정책)까지 챙겨야 할 것들이 한두 가지가 아니었다. 이 글은 그 조사 결과를 정리한 학습 노트다.

## 리전부터 확인해야 하는 이유

OCI Generative AI(GenAI) 서비스는 현재 특정 리전에서만 제공된다. 2026년 기준 US Midwest(Chicago), Germany Central(Frankfurt) 등 일부 리전이 주요 앵커다. **한국 리전(`ap-seoul-1`, `ap-chuncheon-1`)에는 GenAI 서비스가 직접 제공되지 않는다.**

교육 데이터는 학습자 개인정보를 포함한다. 데이터가 미국이나 유럽 리전을 거치면 세 가지 문제가 생긴다:

- 개인정보보호법상 국외 이전 동의 절차 필요
- ISMS-P 심사 시 데이터 경로 소명 요구
- 레이턴시(Latency, 응답 지연) 증가

해결 경로는 두 갈래다. GenAI 서비스를 쓰되 국외 이전 절차를 명확히 밟거나, OCI Data Science에서 오픈소스 모델을 한국 리전에 직접 배포하는 것. 교육 사업의 특성상 후자가 훨씬 안전할 때가 많다.

## 파인튜닝 경로: 관리형 vs 오픈모델 직접 튜닝

OCI에서 파인튜닝(Fine-tuning, 사전학습된 LLM을 특정 도메인 데이터로 추가 학습하는 기법) 경로는 크게 두 갈래다.

### 관리형 GenAI 파인튜닝

OCI Generative AI가 지원하는 모델(Cohere Command 계열 등)을 콘솔에서 파인튜닝할 수 있다. JSONL 포맷으로 데이터를 업로드하면 된다:

```jsonl
{"prompt": "다음 학생 에세이를 채점하시오: ...", "completion": "점수: 4/5, 근거: ..."}
{"prompt": "...", "completion": "..."}
```

인프라 관리가 필요 없다는 게 장점이다. 단점은 앞서 말한 리전 제약과, 학습 데이터가 OCI 관리 인프라를 거친다는 점이다. 데이터 거버넌스(Data Governance, 데이터 관리·보호·책임 체계) 측면에서 민감한 교육 데이터라면 한 번 더 따져봐야 한다.

### OCI Data Science + 오픈모델 LoRA 튜닝

한국 리전에 OCI Data Science 환경을 구성하고 Llama, EXAONE 같은 오픈소스 모델을 직접 튜닝하는 방법이다. LoRA(Low-Rank Adaptation, 대형 언어 모델의 일부 가중치만 학습해 GPU 메모리를 절약하는 파인튜닝 기법)를 쓰면 GPU 부담을 크게 줄일 수 있다:

```python
from peft import LoraConfig, get_peft_model

lora_config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "v_proj"],
    lora_dropout=0.05,
)
model = get_peft_model(base_model, lora_config)
```

데이터가 한국 리전 밖으로 나가지 않으므로 데이터 레지던시 요건 충족이 쉽다. 대신 GPU 인스턴스 비용과 MLOps 관리 부담이 따른다.

## RAG 구성: 벡터 DB부터 한국어 분석기까지

파인튜닝만으로는 부족하다. 교육 LLM Agent는 최신 교육과정, 문제 풀이 레퍼런스, 채점 기준을 실시간으로 참조해야 한다. 이게 RAG(Retrieval-Augmented Generation, 외부 지식을 검색해 LLM 응답에 활용하는 기법)가 필요한 이유다.

OCI 스택 기준 RAG 구성 요소를 정리하면:

| 구성 요소 | OCI 옵션 | 주요 고려사항 |
|---|---|---|
| 벡터 DB | Oracle DB 23ai AI Vector Search | HNSW 인덱스, COSINE 유사도 지원 |
| 전문 검색 | OCI Search with OpenSearch | 한국어 nori 분석기 설정 필수 |
| 임베딩 모델 | OCI Model Catalog 또는 자체 배포 | 리전 가용성 사전 확인 |
| 감사 추적 | OCI Audit + Logging | 교육 데이터 접근 로그 관리 |

Oracle DB AI Vector Search는 SQL과 벡터 검색을 동시에 쓸 수 있다:

```sql
SELECT doc_id, content,
       VECTOR_DISTANCE(embedding, :query_vec, COSINE) AS score
FROM edu_documents
ORDER BY score
FETCH FIRST 5 ROWS ONLY;
```

OpenSearch를 한국어 문서 검색에 쓴다면 nori 분석기(한국어 형태소 분석기) 설정이 반드시 필요하다:

```json
{
  "settings": {
    "analysis": {
      "analyzer": {
        "korean": {
          "type": "custom",
          "tokenizer": "nori_tokenizer",
          "filter": ["nori_part_of_speech"]
        }
      }
    }
  }
}
```

nori 없이 기본 분석기를 쓰면 한국어 조사·어미 처리가 제대로 안 돼 검색 품질이 눈에 띄게 떨어진다.

## 운영 관리: Audit·Model Catalog·Monitoring

교육 AI 사업에서 데이터 레지던시는 기술 문제가 아니라 사업 리스크다. 개인정보보호위원회의 국외 이전 기준, 발주처 보안 요구사항이 모두 얽혀 있다. OCI에서 이를 관리하는 도구 세 가지를 처음 설계 단계부터 넣어야 한다:

- **OCI Audit**: 누가 언제 어떤 리소스에 접근했는지 자동 기록. 나중에 감사 대응 시 필수.
- **OCI Model Catalog**: 어떤 모델을 어느 시점에 배포했는지 버전 관리. 모델 롤백 시 기준점이 된다.
- **OCI Monitoring + Alarms**: 모델 응답 품질 저하나 이상 트래픽을 실시간 감지.

이 세 가지를 초기 아키텍처에서 빠뜨리면, 나중에 감사 대응이나 장애 분석 시 손쓸 방법이 없다.

## 마치며

OCI 기반 교육 LLM Agent를 설계하면서 가장 크게 배운 것은 하나다. **기술 스택 선택보다 리전·데이터 거버넌스가 먼저다.**

파인튜닝 코드는 나중에 바꿀 수 있다. 하지만 데이터가 이미 해외 리전을 거쳐버렸다면 그 리스크는 코드로 되돌릴 수 없다. 구현을 시작하기 전에 서비스 가용 리전 확인 → 데이터 국외 이전 여부 판단 → 감사 추적 설계 순서를 먼저 끝내는 것이 맞는 순서라는 걸 이번 조사에서 확실히 깨달았다.
```