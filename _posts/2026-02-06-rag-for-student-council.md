---
title: "학생회 데이터에 최적화된 RAG — 실전 노하우"
date: 2026-02-06T02:10:00+09:00
categories: [AI, Practice]
tags: [RAG, 학생회, AssoAI, 검색최적화, 할루시네이션]
toc: true
toc_sticky: true
excerpt: "제휴, 행사, 예산, 회의록 — 학생회 데이터는 일반 RAG로는 커버가 안 됩니다. AssoAI가 실전에서 터득한 멀티소스 검색 전략과 할루시네이션 방지 기법을 공유합니다 🐧"
---

학생회 데이터는 기업 데이터와 성격이 다릅니다. 데이터양은 적지만 종류가 다양하고, 한 질문에 여러 테이블을 넘나들어야 합니다. AssoAI가 실전에서 부딪히고 해결한 노하우를 공유합니다 🐧

## 학생회 데이터의 특성

| 데이터 유형 | 특성 | 저장 방식 |
|---|---|---|
| **제휴 정보** | 파트너사, 계약 기간, 수수료율 | DB (partnerships + partners) |
| **행사 일정** | 이름, 날짜, 장소, 예산 | DB (events) |
| **예산/재정** | 수입·지출, 카테고리별 분류 | DB (financial_transactions) |
| **회의록** | 전문 텍스트, AI 요약 | DB + 벡터 검색 |
| **규정/회칙** | 조항별 구조화 텍스트 | DB + 벡터 임베딩 |
| **업로드 문서** | 계약서, 가이드, 체크리스트 | 파일 임베딩 (벡터) |
| **Notion** | 위키, 노트, 데이터베이스 | Notion API + 벡터 |

**핵심 난제**: "지난 학기 제휴 업체 만료된 거 알려줘"라는 질문 하나에 partnerships(제휴 상태) + partners(업체 정보) + 날짜 계산이 동시에 필요합니다.

## 멀티소스 검색 전략

AssoAI의 검색은 3가지 **패스(path)**가 있습니다:

### 패스 1: VIEW 고속 경로

```typescript
// "이번 달 지출 얼마야?" → SQL 생성 없이 바로 VIEW 조회
const matchedView = matchView(query) // → 'v_current_month_expenses'
const result = await queryView(matchedView, orgId)
```

8개의 미리 정의된 VIEW가 가장 흔한 질문 패턴을 커버합니다. LLM SQL 생성(~1초)을 건너뛰므로 체감 속도가 크게 빨라집니다.

### 패스 2: NL→SQL 동적 쿼리

VIEW에 매칭되지 않으면 LLM이 PostgreSQL을 생성합니다. `date-parser`가 한국어 날짜 표현을 해석합니다:

```typescript
parseDatePeriod("지난 달")
// → { startDate: 2026-01-01, endDate: 2026-01-31,
//     periodType: 'month', periodLabel: '지난 달' }

parseDatePeriod("최근 3주")
// → { startDate: 2026-01-16, endDate: 2026-02-06,
//     periodType: 'custom', periodLabel: '최근 3주' }
```

"지난 분기", "작년", "최근 N일" — 한국어 시간 표현 20여 가지를 커버합니다.

### 패스 3: 하이브리드 벡터 검색

문서·Notion·규정은 벡터 유사도 + 전문 검색(Full-Text Search)을 **병렬 실행** 후 병합합니다:

```typescript
// Notion 하이브리드 검색
const [vectorResults, fullTextResults] = await Promise.all([
  searchNotionEmbeddings(query, orgId, limit),  // 의미 검색
  searchNotionFullText(query, orgId, limit),     // 키워드 검색
])
// 중복 제거 후 병합 (벡터 우선)
```

의미적 유사성(벡터)과 정확한 키워드 매칭(FTS)을 동시에 활용하면 recall이 크게 올라갑니다.

## 컨텍스트 윈도우 최적화

학생회 데이터는 대부분 짧고 구조적이라, 토큰을 아끼면서 정보 밀도를 극대화할 수 있습니다:

**1. 계층적 컨텍스트 구성**

```
[스키마 정보]     → LLM이 데이터 구조를 이해
[핵심 데이터]     → 질문에 직접 관련된 VIEW/SQL 결과 (최대 2개)
[보조 데이터]     → 문서 검색 결과 (참고용)
[인용 목록]       → [1], [2] 형식 출처 참조
[응답 지침]       → 형식 규칙 + 할루시네이션 방지
```

**2. DB 결과의 스마트 포매팅**

```typescript
// UUID, org_id, created_at 등 기술 필드를 숨기고
// 금액은 한국어 형식, 날짜는 한국어 로케일로 변환
formatResultsAsContext('events', results)
// → "1. 신입생 환영회 [PLANNED] 500,000원 - 2026년 3월 15일"
```

기술 필드를 제거하면 같은 정보를 **~40% 적은 토큰**으로 전달할 수 있습니다.

## 할루시네이션 방지 — 3중 방어

### 방어 1: 검색 품질 게이트

신뢰도 0.3 미만이면 답변 생성을 차단합니다:

```typescript
if (!quality.hasSufficientEvidence) {
  return generateInsufficientEvidenceResponse(query, quality)
  // → "근거 부족으로 정확한 답변이 어렵습니다."
}
```

### 방어 2: "데이터 없음"도 유효한 답변

DB 쿼리가 성공적으로 실행됐지만 0건이면, 신뢰도 0.5로 **"해당 데이터가 없습니다"**라고 답합니다. "모른다"와 "없다"를 구분하는 것이 핵심입니다.

```typescript
if (dbQueryExecuted && totalResults === 0) {
  return {
    confidence: 0.5,
    hasSufficientEvidence: true, // "없다"는 유효한 답변!
  }
}
```

### 방어 3: 프롬프트 레벨 강제

```
필수 규칙:
1. 위 데이터에 있는 정보만 사용하세요
2. 데이터에 없는 정보는 절대 추측하지 마세요
3. 불확실하면 "해당 정보를 확인할 수 없습니다"라고 답하세요
```

## 실제 쿼리 흐름 예시

**질문**: "지난 학기 제휴 업체 만료된 거 알려줘"

```
1. context-synthesizer → 대화 맥락 확인
2. query-analyzer → {
     intent: 'search',
     dataSource: 'structured',
     relevantTables: ['partnerships', 'partners'],
     keywords: ['제휴', '만료', '지난 학기']
   }
3. date-parser → "지난 학기" 해석 실패 → NL→SQL로 폴백
4. db-query → LLM이 SQL 생성:
   SELECT p.name, ps.title, ps.status, ps.end_date
   FROM partnerships ps
   JOIN partners p ON ps.partner_id = p.id
   WHERE ps.org_id = '{orgId}'
     AND ps.status = 'EXPIRED'
     AND ps.end_date >= '2025-09-01'
   ORDER BY ps.end_date DESC
5. reranker → (DB 결과라 벡터 리랭킹 불필요)
6. context-builder → 프로덕션 컨텍스트 조립
7. LLM → "지난 학기에 만료된 제휴는 3건입니다: ..."
```

전체 과정에 **~2초**. 사용자는 ThinkingStep SSE로 진행 상황을 실시간으로 볼 수 있습니다.

## 교훈

1. **흔한 질문은 패턴으로** — VIEW 매칭이 80%의 질문을 커버
2. **NL→SQL은 보안 필수** — org_id 강제, SELECT만 허용
3. **"없다"와 "모른다"를 구분** — 할루시네이션 방지의 핵심
4. **한국어 날짜 파서** — "지난 분기", "최근 N일" 같은 표현을 직접 파싱
5. **벡터+FTS 하이브리드** — 의미 검색과 키워드 검색을 동시 활용

학생회 데이터는 작지만 복잡합니다. 범용 RAG로는 한계가 있고, 도메인에 맞는 최적화가 정확도를 결정합니다 🐧

---

*AssoAI는 학생회·동아리를 위한 AI 운영 도구입니다. [asso-ai.kr](https://asso-ai.kr)*
