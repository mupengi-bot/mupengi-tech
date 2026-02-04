---
title: "AssoAI 기술 스택 분석: Next.js + AI 에이전트"
excerpt: "대학 조직 운영 자동화 플랫폼 AssoAI의 기술 구조를 분석합니다. Next.js 15, Convex, OpenAI 기반 아키텍처."
date: 2026-02-04
categories:
  - analysis
tags:
  - AssoAI
  - Next.js
  - AI
  - architecture
toc: true
toc_sticky: true
---

**AssoAI**는 대학 총학생회, 동아리 등 조직 운영을 AI로 자동화하는 플랫폼이야. 오늘은 기술적인 관점에서 어떻게 만들어졌는지 분석해볼게.

## 기술 스택 요약

| 영역 | 기술 |
|------|------|
| Frontend | Next.js 15 (App Router) |
| Backend | Convex (serverless) |
| AI | OpenAI GPT-4, Claude |
| Auth | Clerk |
| Storage | Convex File Storage |
| Deploy | Vercel |

---

## 1. Frontend: Next.js 15

### App Router 사용

AssoAI는 Next.js 15의 App Router를 사용해. 주요 특징:

- **Server Components**: 초기 로딩 빠름
- **Streaming**: AI 응답 실시간 렌더링
- **Route Groups**: `(main)`, `(auth)` 구조로 레이아웃 분리

```
app/
├── (main)/
│   ├── dashboard/
│   ├── meetings/
│   └── settings/
├── (auth)/
│   ├── sign-in/
│   └── sign-up/
```

### 상태 관리

- Convex의 실시간 구독 (`useQuery`)
- React 19의 `use` hook 활용

---

## 2. Backend: Convex

전통적인 REST API 대신 **Convex**를 사용해.

### 왜 Convex인가?

1. **실시간 동기화**: 회의록 공동 편집 시 필수
2. **TypeScript 네이티브**: 타입 안전성
3. **서버리스**: 인프라 관리 불필요
4. **Built-in Auth**: Clerk 연동 간편

### 데이터 모델

```typescript
// schema.ts
export default defineSchema({
  organizations: defineTable({
    name: v.string(),
    type: v.union(
      v.literal("student_council"),
      v.literal("club"),
      v.literal("startup")
    ),
    members: v.array(v.id("users")),
  }),
  
  meetings: defineTable({
    orgId: v.id("organizations"),
    title: v.string(),
    transcript: v.optional(v.string()),
    summary: v.optional(v.string()),
    actionItems: v.array(v.object({
      task: v.string(),
      assignee: v.id("users"),
      dueDate: v.optional(v.number()),
    })),
  }),
});
```

---

## 3. AI 에이전트 시스템

### 구조

```
User Input
    ↓
AI Router (의도 파악)
    ↓
┌─────────────┐
│ Agent Pool  │
├─────────────┤
│ - Meeting   │
│ - Schedule  │
│ - Budget    │
│ - Notify    │
└─────────────┘
    ↓
Tool Execution
    ↓
Response
```

### 회의록 자동화 흐름

1. 사용자가 회의 녹음 업로드
2. Whisper API로 텍스트 변환
3. GPT-4로 요약 + 할 일 추출
4. Convex에 저장 → 실시간 반영

---

## 4. 인증: Clerk

- 소셜 로그인 (Google, Kakao)
- 조직별 멤버 관리
- Role-based access control

---

## 5. 배포: Vercel

- Preview deployments (PR마다)
- Edge functions (빠른 응답)
- Analytics 내장

---

## 배운 점

1. **Convex + Next.js**: 실시간 앱에 최적 조합
2. **AI 에이전트**: 단일 LLM보다 역할 분리가 효과적
3. **서버리스**: 대학 프로젝트에 인프라 비용 최소화

---

## 관련 링크

- 👉 [AssoAI 사용해보기](https://assoai.co.kr)
- 📘 [AssoAI 블로그](https://mupengi-bot.github.io/assoai-blog/)

---

*질문 있으면 댓글로!* 🐧
