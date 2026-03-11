---
title: "AI 모델사가 에이전트 프레임워크를 통합하는 아키텍처 패턴"
date: 2026-03-11
categories: [AI Architecture]
tags: [agent-framework, multi-agent, openclaw, architecture, llm]
excerpt: "모델 API만으로는 부족하다. AI 모델 제공사가 오픈소스 에이전트 프레임워크를 자사 서비스에 통합하는 구조를 분석한다."
toc: true
---

## 모델 API만으로는 부족한 시대

2026년 현재, LLM 모델 API를 제공하는 것만으로는 차별화가 어렵다. 사용자는 "똑똑한 챗봇"이 아니라 **실제로 일을 처리하는 에이전트**를 원한다. 파일을 읽고, 코드를 실행하고, 외부 서비스를 호출하는 — 그런 능력.

이 격차를 메우기 위해 모델사들이 택하는 전략이 있다: **오픈소스 에이전트 프레임워크 통합**.

## 통합 아키텍처의 3계층

모델사가 에이전트 프레임워크를 통합할 때 보이는 공통 구조가 있다.

```
┌─────────────────────────────┐
│   Application Layer         │  ← 사용자 UI (채팅, 사이드바)
├─────────────────────────────┤
│   Agent OS Layer            │  ← 에이전트 프레임워크
│   (도구 호출, 메모리, 스킬) │     (OpenClaw, LangChain 등)
├─────────────────────────────┤
│   Model Layer               │  ← LLM API
│   (추론, 생성)              │     (GPT, Claude, Kimi 등)
└─────────────────────────────┘
```

핵심은 **Agent OS Layer**가 모델과 분리되어 있다는 점이다. 모델은 교체 가능하고, 에이전트의 도구·메모리·컨텍스트는 프레임워크가 관리한다.

## 실제 통합 패턴: 사이드바 임베딩

최근 중국의 한 AI 모델사가 자사 채팅 서비스 사이드바에 OpenClaw 기반 에이전트를 통합했다. 구조를 보면:

```python
# 간소화된 통합 흐름
class AgentSidebarService:
    def __init__(self):
        self.model_client = ModelAPI()        # 자사 LLM
        self.agent_runtime = OpenClawRuntime() # 에이전트 프레임워크
    
    def handle_request(self, user_input: str):
        # 1) 에이전트가 의도 분석 + 도구 선택
        plan = self.agent_runtime.plan(user_input)
        
        # 2) 도구 실행 (파일 읽기, 웹 검색, 코드 실행 등)
        tool_results = []
        for step in plan.steps:
            result = self.agent_runtime.execute_tool(step)
            tool_results.append(result)
        
        # 3) 모델이 최종 응답 생성
        response = self.model_client.generate(
            context=tool_results,
            prompt=user_input
        )
        return response
```

이 구조의 장점:

- **모델사**: 자사 모델의 활용 범위가 "채팅"에서 "에이전트"로 확장
- **프레임워크**: 대규모 유저 베이스 확보
- **사용자**: 별도 설치 없이 에이전트 기능 사용

## 모델 종속 vs 모델 중립

여기서 아키텍처 선택이 갈린다:

```yaml
# 모델 종속 통합 (Model-Locked)
agent:
  model: kimi-k2.5  # 자사 모델만 사용 가능
  tools: [search, code, file]
  memory: cloud-only

# 모델 중립 통합 (Model-Agnostic)
agent:
  model: ${USER_CHOICE}  # Claude, GPT, Kimi 등 선택
  tools: [search, code, file, custom_skills]
  memory: local + cloud
```

모델사 입장에서는 **종속 통합**이 유리하다 — API 호출량이 보장되니까. 하지만 사용자 입장에서는 **모델 중립**이 낫다. 모델을 바꿔도 에이전트의 메모리와 스킬이 유지되기 때문이다.

## 멀티에이전트로의 확장

단일 에이전트 통합은 시작일 뿐이다. 실전에서는 여러 에이전트가 협업하는 구조가 필요하다:

```python
# 서브에이전트 오케스트레이션
class AgentOrchestrator:
    def __init__(self, max_parallel=6):
        self.agents = {}
        self.max_parallel = max_parallel
    
    def spawn(self, task: str, role: str):
        """1 에이전트 = 1 미션"""
        agent = SubAgent(role=role, task=task)
        agent.start()
        self.agents[agent.id] = agent
        return agent.id
    
    def collect_results(self):
        results = {}
        for aid, agent in self.agents.items():
            if agent.done:
                results[aid] = agent.result
        return results
```

이때 중요한 건 **컨텍스트 격리**다. 서브에이전트는 메인 에이전트의 전체 메모리에 접근하면 안 된다. 필요한 컨텍스트만 전달하고, 결과만 수집한다.

## 시사점

1. **모델 ≠ 제품**: API만으로는 제품이 안 된다. 에이전트 레이어가 필수.
2. **프레임워크가 해자**: 모델은 빠르게 동질화되지만, 에이전트의 메모리·스킬·컨텍스트는 쌓일수록 강해진다.
3. **오픈소스가 표준화**: GitHub 200K+ stars를 받는 프레임워크가 사실상 표준이 되고, 모델사가 여기에 올라타는 구조.

AI 에이전트 시대에서 진짜 경쟁은 "누가 더 똑똑한 모델을 만드느냐"가 아니라 **"누가 더 좋은 에이전트 OS를 만드느냐"**로 이동하고 있다.
