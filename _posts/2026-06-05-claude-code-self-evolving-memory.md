---
title: "Claude Code 를 잊지 않는 에이전트로 — Reflector-Curator 메모리 패턴"
date: 2026-06-05 23:50:00 +0900
categories: [AI, LLM]
tags: [Claude, Hook, 메모리, 자동화, Gemma, NotebookLM]
---

## 매번 같은 설명을 또 했다

Claude Code 를 매일 쓰면서 점점 답답해진 게 있다. **새 세션이 시작될 때마다 "나는 누구고, 우리 코드는 어떤 컨벤션을 쓰고, 어제 무슨 결정을 내렸는지" 처음부터 다시 설명해야 한다는 것.** 컨텍스트가 휘발한다. 결국 같은 피드백 ("이 컨벤션은 X 야") 을 한 달에 다섯 번씩 주고 있었다.

`CLAUDE.md` 에 적어두면 되긴 하지만, 그건 **선언적 규칙**만 잡지 **시간이 흐르며 쌓이는 결정·실수·취향**은 못 잡는다. 그래서 "잊지 않는 에이전트" 를 만들기로 했다.

## 영감 — Archon NotebookLM 의 Reflector-Curator

NotebookLM 으로 "deterministic AI coding harness 의 메모리 패턴" 을 deep research 돌리다 발견한 키워드가 **Reflector-Curator** 였다. 핵심은:

- **Reflector** — 세션이 끝나면 transcript 전체를 다시 읽고, 영구 보존 가치 있는 항목만 추출. evidence 인용을 강제해서 환각 방지.
- **Curator** — 추출 결과를 기존 메모리와 비교해 add / update / delete / skip 중 결정.
- **Evidence Gate** — "I believe", "should be" 같은 표현 금지. 트랜스크립트 내 정확한 메시지 ID·파일 경로 인용 필수.
- **요약 표류 회피** — 이전 요약을 다시 요약하는 incremental 방식은 오차 누적. 항상 원본에서 재생성.

이걸 Claude Code 의 hook 시스템 위에 얹기로 했다.

## Hook chain 으로 자기 진화 만들기

Claude Code 는 `~/.claude/settings.json` 에 lifecycle hook 을 등록할 수 있다. 우리가 활용한 건 다섯 개:

```
PreToolUse   → 모든 도구 호출 감시 (events.jsonl emit)
PostToolUse  → 결과 캡처
SessionStart → 메모리 컨텍스트 inject
Stop         → 매 응답 끝 (가벼움, 폭주 방지로 finalizer 안 부름)
PreCompact   → 진짜 세션 reset 시점 (여기서 모든 무거운 작업)
```

세션이 길어서 자동 압축이 일어나면 PreCompact 가 발동한다. 그 한 번의 발동 안에서 다음이 일제히 돈다:

```
PreCompact
  ├─ Gemma (로컬) — 한 줄 요약을 프로젝트 MEMORY.md 에 append
  ├─ Gemma (로컬) — 디테일한 bullet 을 daily_log/<날짜>.md 에 append (h3 + 시간)
  ├─ Claude Sonnet (Reflector) — transcript 분석, 9 카테고리 추출
  ├─ Claude Sonnet (Curator)   — extraction → mutation 결정
  └─ Gemma (로컬) — raw memory 전체를 압축한 _compiled_memory.md 갱신
```

## 모델 분리가 핵심

처음엔 모든 단계를 Claude Sonnet 으로 했다가 quota 가 빠르게 다 찼다. 그래서 **역할 분리**:

| 작업 | 모델 | 이유 |
|---|---|---|
| 의미 판단 (Reflector / Curator) | Claude Sonnet | 정확도 |
| 요약 / 압축 / 분류 | Gemma 4 26B (로컬 Ollama) | 무료·무제한 |
| 시작 시 컨텍스트 inject | 미리 compile 된 markdown | 추가 호출 0 |

Gemma 호출 시 한 가지 함정 — `think: true` 옵션을 켜면 thinking 이 `num_predict` 토큰 예산을 다 먹어서 응답이 비어 나온다. JSON 출력 받으려면 `think: false` 가 답.

## 2-레이어 메모리 주입

SessionStart 에서 두 곳의 `_compiled_memory.md` 를 inject 한다:

1. **Global** — 모든 프로젝트 공통 컨벤션 (코드 스타일, 금기, 사용자 선호)
2. **Project-specific** — 이 프로젝트 한정 결정·핸드오프

```
~/.claude/projects/-Users-x-Desktop-vault/memory/          ← Global
~/.claude/projects/-Users-x-Desktop-vault-01-proj-a/memory/ ← Project-specific
```

한 곳에 새 규칙을 박으면 모든 프로젝트 세션에 자동 반영된다. 예: "파일 최상단에 docstring/overview 주석 절대 금지" 같은 규칙을 global 에 한 번 적으면, 어느 프로젝트에서 새 .py 만들든 Claude 가 그 규칙을 따른다.

## 비용 폭주 방지 장치

Claude Code 의 `Stop` hook 은 **매 응답 끝마다** 발동한다. 긴 세션이면 100회 넘게 호출된다. 처음에 finalizer 를 Stop 에 걸었더니 100번 finalizer 가 돌아서 API quota 가 1시간 만에 다 찼다.

해결:

- Finalizer 트리거를 **`PreCompact` 한 곳에만** — 진짜 세션 reset 시점에서만 무거운 작업
- **`session_id` lockfile** — 동일 세션의 hook 이 겹쳐 호출돼도 1회만 실행
- **`HARNESS_CHILD_HOOK=1`** 환경변수 — 우리 스크립트가 spawn 한 자식 프로세스가 다시 hook 체인을 타면 즉시 exit. 재귀 폭주 차단

## 결과

지금 38개의 raw memory 파일이 누적됐다. 시작 시 5KB 짜리 압축 디지스트가 주입되고, 거기에 다음과 같은 게 적혀있다:

- 사용자 선호 (한국어 답변, 품질 우선)
- 절대 규칙 (모듈 docstring 금지, conda env 컨벤션)
- 진행 중 프로젝트 상태
- 과거 실수 패턴 (CSS specificity 충돌, Marp stdin hang, launchd EPERM ...)

매 세션 시작 시 Claude 가 이걸 보고 동작한다. **같은 피드백 두 번 안 줘도 되는 환경**이 만들어졌다.

## 다음 단계

- **FTS5 회상** — SQLite full-text 인덱스로 과거 transcript 검색. "전에 그거 어떻게 고쳤지?" 시 on-demand
- **Skill 자율 진화** — 어려운 문제 풀면 `SKILL.md` 자동 작성 (Gemma 가 감지, Claude 가 작성)
- **Hermes Agent sidecar** — Discord/Slack 라우팅·multi-profile

## 마치며

이 블로그 글 자체도 위 시스템 덕에 빠르게 정리됐다. 매일 작업하면서 쌓인 daily_log 와 결정 기록 (ADR) 에서 자동으로 draft 가 만들어지고, 나는 검토만 한다.

다음 글에서는 FTS5 인덱스로 "지난번 그거" 같은 자연어 회상을 어떻게 구현했는지 다룬다.
