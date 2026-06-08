---
title: "스스로 진화하는 LLM Agent 논문 3편에서 보인 공통 패턴"
date: 2026-06-08 17:00:00 +0900
categories: [AI]
tags: [LLM-Agent, Self-Evolving, Multi-Agent]
---

# 도입 (왜 이 글)

최근 MLEvolve, Adaptive Auto-Harness, EvoDS 세 논문을 연달아 읽었다. 각 논문이 다루는 과제는 달랐지만 — 코드 생성, 자동화 테스트 하네스 구축, 데이터셋 진화 — 읽고 나서 묘하게 같은 냄새가 났다. "agent가 경험을 쌓아 스스로 나아진다"는 구조적 패턴이 세 편 모두에 반복됐다.

이 글은 논문 요약이 아니다. 세 편을 겹쳐 놓고 보니 드러난 **공통 설계 문법**을 내 언어로 정리한 노트다.

---

## 패턴 1 — Memory 없이는 진화도 없다

세 논문 중 어느 것도 "매번 처음부터 추론하는" 구조를 쓰지 않는다. 공통적으로 **경험을 외부 스토리지에 저장하고 다음 스텝에서 검색해 재사용**한다.

MLEvolve는 성공한 코드 변환 패턴을 벡터 DB에 누적한다. Adaptive Auto-Harness는 이전 실행에서 실패한 케이스를 harness tree(테스트 실행 트리)에 기록해 다음 생성 시 회피 신호로 쓴다. EvoDS는 데이터 품질 점수를 feedback loop으로 돌려 SFT(Supervised Fine-Tuning) 데이터를 점진적으로 정제한다.

핵심은 단순 로그가 아니라는 점이다. 경험이 **검색 가능하고, 다음 결정에 영향을 줄 수 있는 형태**로 저장되어야 진화로 이어진다. 기억하지 못하는 agent는 반복만 할 뿐이다.

```
경험 저장 흐름 (세 논문 공통 추상)
실행 → 결과 평가 → 벡터/트리/DB 저장 → 다음 실행 시 유사 경험 검색 → prompt 보강
```

---

## 패턴 2 — 작업이 다양해지면 routing이 필요해진다

단일 prompt로 모든 작업을 처리하는 구조는 작업 범위가 커질수록 한계가 온다. 세 논문은 각자 다른 방식으로 이 문제를 해결했지만, 방향은 같다 — **작업의 성격에 따라 다른 경로로 분기시킨다.**

Adaptive Auto-Harness가 가장 직관적이다. 입력 과제를 분류한 뒤 그에 맞는 편집 전략을 선택한다:

```
Full rewrite     ← 기존 코드와 연관성 낮은 새 요구사항
Stepwise         ← 단계별로 검증이 필요한 복잡한 변환
Diff-based       ← 기존 코드 베이스를 보존하며 일부만 수정
```

MLEvolve는 `Analyze → Research → Build → Verify` 파이프라인으로 단계를 명시적으로 분리한다. 각 단계가 사실상 specialized sub-agent처럼 동작한다. EvoDS는 데이터 유형에 따라 생성 전략을 달리 적용한다.

공통 교훈: **routing은 성능 최적화가 아니라 신뢰성 확보 수단**이다. 잘못된 전략을 잘못된 작업에 붙이면 아무리 강력한 모델이어도 결과가 불안정해진다.

---

## 패턴 3 — Long-horizon 작업의 병목은 skill과 context

짧은 one-shot 작업에서는 보이지 않던 문제가 장기 실행 agent에서 두드러진다. **skill acquisition**(기술 습득)과 **context compression**(문맥 압축)이다.

MLEvolve와 EvoDS 모두 초기에는 범용 전략으로 시작하지만, 실행이 누적될수록 특정 도메인에 특화된 패턴 — 사실상 skill — 이 memory에 축적된다. 이 skill을 얼마나 효율적으로 검색해 재사용하느냐가 장기 성능의 분기점이 된다.

context compression은 더 현실적인 병목이다. 실행 이력이 길어질수록 prompt가 token 한계에 부딪힌다. EvoDS는 GRPO(Group Relative Policy Optimization, 그룹 상대 정책 최적화)로 모델 자체를 반복 fine-tuning해 이 문제를 우회한다. Adaptive Auto-Harness는 harness tree를 가지치기해 context를 정리한다.

```
SFT → GRPO  ← EvoDS의 progressive self-improvement 루프
             (SFT로 초기 학습, GRPO로 정책 강화)
```

두 문제는 사실 같은 원인에서 온다 — **agent가 오래 살수록 무엇을 기억하고 무엇을 버릴지 판단해야 한다.**

---

## 마치며

세 논문을 읽기 전에는 "self-evolving agent"가 그냥 피드백 루프를 추가한 것쯤으로 생각했다. 읽고 나니 훨씬 구체적인 설계 결정들이 있었다 — memory 구조, routing 정책, skill 저장 방식, context 관리 전략.

당장 프로덕션 agent를 만들 계획이 없더라도, 이 패턴들은 단일 LLM 호출을 엮어 조금 더 복잡한 자동화를 만들 때도 그대로 적용된다. "이 결과를 어떻게 저장하고, 다음에 어떻게 꺼낼 것인가"라는 질문이 생기는 순간, 이미 그 안에 있는 셈이다.