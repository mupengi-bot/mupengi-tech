---
title: "AssoAI RAG 아키텍처 딥다이브 — 18개 모듈의 비밀"
date: 2026-02-06T02:00:00+09:00
categories: [AI, Architecture]
tags: [RAG, AssoAI, LLM, Vector Search, TypeScript]
toc: true
toc_sticky: true
excerpt: "학생회 SaaS AssoAI의 RAG 파이프라인을 코드 레벨에서 해부합니다. 18개 모듈이 어떻게 협력하여 쿼리→분석→검색→리랭킹→합성→응답의 여정을 완성하는지 살펴보아요 🐧"
---

학생회 SaaS **AssoAI**의 심장부에는 18개 TypeScript 모듈로 이루어진 RAG(Retrieval-Augmented Generation) 시스템이 뛰고 있습니다. "이번 달 예산 얼마 남았어?"라는 한 줄 질문이 정확한 답변으로 돌아오기까지, 내부에서는 놀라울 만큼 정교한 파이프라인이 작동합니다. 오늘은 그 여정을 코드 레벨에서 낱낱이 해부해 보겠습니다 🐧

## 전체 아키텍처 한눈에 보기

```
사용자 질문
    │
    ▼
┌─────────────────────────────────────────────┐
│  1. context-synthesizer  ← 대화 맥락 합성    │
│  2. query-analyzer       ← 의도·키워드 분석   │
│  3. date-parser          ← 날짜 표현 파싱     │
├─────────────────────────────────────────────┤
│          검색 레이어 (병렬 실행)               │
│  4. view-query     ← VIEW 패턴 매칭          │
│  5. db-query       ← NL→SQL 변환·실행        │
│  6. Vector Search  ← 임베딩 유사도 검색       │
│  7. notion-search  ← Notion 하이브리드 검색   │
│  8. regulation-search ← 규정 벡터+텍스트      │
│  9. comprehensive-search ← 멀티소스 통합      │
├─────────────────────────────────────────────┤
│  10. reranker           ← 중복 제거 + 재점수  │
│  11. schema-info        ← DB 스키마 제공      │
│  12. context-builder    ← 프로덕션 컨텍스트   │
│  13. answer-enhancer    ← 인용·링크 생성      │
├─────────────────────────────────────────────┤
│  14. action-planner     ← 액션 의도 감지      │
│  15. action-executor    ← DB 쓰기 + 실행취소  │
│  16. follow-up-generator← 후속 질문 추천      │
├─────────────────────────────────────────────┤
│  17. evaluation         ← 30문항 평가셋       │
│  18. evaluation-runner  ← 품질 벤치마크       │
├─────────────────────────────────────────────┤
│       orchestrator.ts  ← 전체 파이프라인 지휘  │
└─────────────────────────────────────────────┘
    │
    ▼
  SSE 스트리밍 응답 (thinking steps + answer)
```

## 핵심 모듈 딥다이브

### 1️⃣ Orchestrator — 파이프라인 지휘자

`orchestrator.ts`의 `RAGOrchestrator` 클래스가 전체 흐름을 10단계로 지휘합니다:

```typescript
export class RAGOrchestrator {
  constructor(options: RAGOptions = {}) {
    this.options = {
      maxIterations: options.maxIterations ?? 3,
      similarityThreshold: options.similarityThreshold ?? 0.5,
      resultsPerQuery: options.resultsPerQuery ?? 5,
      enableReranking: options.enableReranking ?? true,
    }
  }
}
```

각 단계마다 `ThinkingStep`을 생성해 프론트엔드에 실시간 SSE 스트리밍합니다. "질문 분석 중 → 데이터베이스 조회 중 → 결과 분석 중 → 답변 생성 중" — 사용자가 AI의 사고 과정을 지켜볼 수 있죠.

### 2️⃣ Query Analyzer — AI 기반 의도 분석

LLM에게 사용자 질문을 JSON으로 구조화하게 합니다:

```typescript
const analysis = await analyzeQuery(effectiveQuery, enhancedContext)
// → { intent, keywords, entities, expandedQueries,
//    dataSource, relevantTables }
```

`dataSource`가 핵심입니다. `"structured"`, `"document"`, `"both"`, `"none"` 중 하나를 반환하여 이후 검색 전략을 결정합니다. 대화형 질문("안녕", "넌 뭘 할 수 있어?")은 `"none"`으로 분류되어 RAG를 완전히 바이패스합니다.

### 3️⃣ Context Synthesizer — 멀티턴 대화 핵심

짧은 후속 질문("숫자로 이야기해줘")을 이전 맥락과 결합합니다:

```typescript
export function isShortFollowUpQuery(query: string): boolean {
  const wordCount = query.trim().split(/\s+/).length
  const hasModifierWords = /숫자로|자세히|더|다시|그거|그것|아까/.test(query)
  return wordCount <= 5 || hasModifierWords
}
```

"그것", "거기서", "아까" 같은 대명사를 실제 대상으로 **해석(resolve)** 하여, 짧은 후속 질문도 독립적인 완전한 쿼리로 확장합니다.

### 4️⃣ View Query — 패턴 매칭 고속 경로

미리 정의된 VIEW 패턴(`v_current_month_expenses` 등 8개)과 키워드를 매칭하여, SQL 생성 없이 Supabase VIEW를 직접 조회합니다:

```typescript
const VIEW_PATTERNS = {
  'v_current_month_expenses': {
    keywords: ['이번 달 지출', '지출 총액', '얼마 썼', ...],
    description: '이번 달 지출 합계',
    resultType: 'single_value',
  },
  // ... 7개 더
}
```

"이번 달 지출 얼마야?"처럼 흔한 질문은 LLM SQL 생성을 건너뛰어 **레이턴시를 수백ms 절약**합니다.

### 5️⃣ DB Query — NL→SQL 변환

VIEW에 매칭되지 않는 구조화 쿼리는 LLM이 PostgreSQL을 생성합니다. **보안이 핵심**입니다:

```typescript
// 위험한 패턴 차단
const dangerousPatterns = [
  /\bINSERT\s+INTO\b/i, /\bDELETE\s+FROM\b/i,
  /\bDROP\s+(TABLE|INDEX|VIEW)\b/i, ...
]
// org_id 필터 강제
if (!response.toLowerCase().includes('org_id')) {
  return null // 쿼리 거부
}
```

SELECT만 허용하고, org_id 필터가 없으면 무조건 거부. 서브쿼리도 3개까지만 허용합니다.

### 6️⃣ Reranker — 4요소 가중 재점수

벡터 검색 결과를 단순 유사도 대신 **4개 요소**의 가중합으로 재정렬합니다:

```typescript
const originalScore = result.similarity * 0.4   // 원본 유사도
const keywordScore = keywordDensity * 0.3        // 키워드 매칭
const positionScore = Math.exp(-chunkIndex*0.1) * 0.2  // 위치 보정
const recencyScore = 0.5 * 0.1                   // 최신성 (향후 확장)
```

Jaccard 유사도 0.8 이상인 중복 청크를 먼저 제거(`deduplicateResults`)하고, 남은 결과를 재점수합니다.

### 7️⃣ Context Builder — 프로덕션 프롬프트 구성

검색된 데이터를 계층적 프롬프트로 조립합니다:

```
[스키마 인식] → [대화 맥락] → [핵심 데이터] → [보조 정보]
→ [사용자 질문] → [인용 목록] → [응답 지침]
```

"데이터에 없는 정보는 절대 추측하지 마세요"라는 **강제 지침**이 할루시네이션 방지의 최후 방어선입니다.

### 8️⃣ Action System — 자연어로 CRUD

"다음 주 수요일 정기회의 잡아줘" → LLM이 구조화 데이터를 추출 → 사용자 확인 → DB 쓰기:

```typescript
const actionPlan: ActionPlan = {
  type: 'create_meeting',
  table: 'meetings',
  operation: 'insert',
  data: { title: '정기회의', meeting_date: '2026-02-11T14:00:00' },
  preview: buildPreview(actionType, extracted),
  requiresConfirmation: true,  // 반드시 사용자 확인!
}
```

`action-executor.ts`는 모든 작업을 **감사 로그**에 기록하고, 30분 내 **실행 취소(undo)**를 지원합니다.

## 검색 품질 평가

`assessSearchQuality`가 검색 결과의 신뢰도를 0~1 스케일로 평가합니다:

```typescript
if (dbQueryExecuted) confidence += 0.4  // 권위 있는 소스
if (dbRowCount > 0)  confidence += 0.2  // 실제 데이터 존재
if (vectorResults >= 1) confidence += 0.2
if (avgSimilarity >= 0.7) confidence += 0.2
```

신뢰도 0.3 미만이면 답변 대신 **"근거 부족"** 메시지를 반환합니다. 모르면 모른다고 하는 것 — 할루시네이션 방지의 핵심 원칙입니다.

## 평가 프레임워크

`evaluation.ts`에 30개 테스트 문항(구조화 15 + 문서 15)이 정의되어 있고, `evaluation-runner.ts`가 자동 벤치마크를 실행합니다:

- **Context Relevance** — 기대 키워드가 검색 결과에 존재하는가?
- **Answer Relevance** — 답변이 질문 유형(숫자/목록/날짜)에 맞는가?
- **Faithfulness** — 답변이 검색된 컨텍스트에 근거하는가?

## 마무리

18개 모듈, 4000줄 이상의 코드가 "이번 달 예산 얼마야?"라는 질문 하나에 동원됩니다. 과하다고요? 하지만 정확성·보안·사용자 경험을 포기하지 않으려면, 이 정도의 정교함이 필요합니다 🐧

다음 포스트에서는 학생회 데이터에 최적화된 실전 노하우를 다루겠습니다!

---

*AssoAI는 학생회·동아리를 위한 AI 운영 도구입니다. [asso-ai.kr](https://asso-ai.kr)*
