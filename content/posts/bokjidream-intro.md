+++
date = '2026-04-19T14:30:21+09:00'
draft = false
title = 'bokjidream — AI 복지 서비스 자가진단 챗봇 프로젝트 소개'
tags = ['bokjidream', 'AI', 'LangGraph', 'RAG', '복지']
series = ['bokjidream']
series_order = 1
summary = '취약계층이 복잡한 복지 제도를 스스로 탐색할 수 있도록 돕는 AI 챗봇 시스템, bokjidream을 소개합니다.'
+++

## 왜 만들었나

한국의 복지 서비스는 종류가 방대하고 신청 절차가 복잡해서, 정작 도움이 필요한 사람들이 자신에게 맞는 서비스를 찾지 못하는 경우가 많습니다. 고령자나 장애인처럼 디지털 접근성이 낮은 분들은 더욱 그렇습니다.

**bokjidream**은 이 문제를 AI로 해결하려는 프로젝트입니다. 사용자가 자신의 상황을 대화로 설명하면, 시스템이 적합한 복지 서비스를 찾아주고 신청 서류 작성까지 도와줍니다.

## 핵심 설계 원칙

- **프라이버시 우선**: 민감한 개인정보를 외부 클라우드에 보내지 않기 위해 **로컬 LLM(Llama 3, Ollama)** 을 사용합니다 (Sovereign AI).
- **최신 복지 정보**: 정부 복지 포털을 직접 크롤링해 RAG 파이프라인으로 검색합니다.
- **단계별 안내**: 단순 검색이 아니라 면담 → 서비스 선정 → 서류 안내 → 초안 작성까지 이어지는 에이전트 흐름을 갖춥니다.

## 시스템 구조

```
사용자 (Next.js UI)
  ↓
LangGraph 멀티 에이전트 (ai/)
  ├─ initial_interview   : 초기 상황 파악
  ├─ rag_search          : 복지 서비스 후보 검색
  ├─ service_select      : 적합 서비스 선정
  ├─ rag_detail          : 상세 정보 수집
  ├─ detail_interview    : 추가 정보 수집
  ├─ document_guidance   : 필요 서류 안내
  ├─ draft_writer        : 신청서 초안 작성
  └─ report_writer       : 최종 리포트 생성
  ↓
로컬 Llama 3 (Ollama)        ←→  RAG 서비스 (rag/)
                                   [Playwright 크롤러
                                    → ChromaDB
                                    → FastAPI /search]
```

8개 노드가 순서대로 실행되며, 각 단계에서 RAG 서비스를 호출해 실시간 복지 정보를 가져옵니다.

## 모노레포 구성

| 디렉토리 | 역할 |
|---------|------|
| `ai/` | LangGraph 에이전트 오케스트레이션 |
| `rag/` | 크롤러 + ChromaDB + FastAPI 검색 API |
| `llm/` | Llama 3 파인튜닝 및 서빙 (계획 중) |
| `web/` | Next.js 프론트엔드 (계획 중) |
| `docs/` | 기획 문서 |

## 현재 상태

현재 LLM은 개발 편의를 위해 **Google Gemini**를 임시로 사용하고 있습니다. 실제 배포 시에는 Ollama 기반 Llama 3으로 전환할 예정이며, 전환 포인트는 `ai/tools/llm.py`의 `get_llm()` 함수 한 곳입니다.

RAG 파이프라인(`rag/`)은 Playwright로 복지 포털을 크롤링하고, 청크를 ChromaDB에 임베딩해 FastAPI로 검색 API를 제공합니다. `ai/` 에이전트는 이 API를 HTTP로 호출합니다.

## 앞으로의 계획

- Llama 3 파인튜닝 (복지 도메인 특화)
- Next.js UI 구현
- 실사용자 대상 파일럿 테스트

다음 포스트에서는 LangGraph 멀티 에이전트 구현 과정을 자세히 다룰 예정입니다.
