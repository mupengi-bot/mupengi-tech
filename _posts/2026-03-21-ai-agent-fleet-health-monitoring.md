---
title: "AI 에이전트 플릿 헬스 모니터링 자동화 — 장애 감지부터 복구까지"
date: 2026-03-21
categories: [AI Agent, DevOps]
tags: [ai-agent, fleet-management, health-monitoring, automation, openclaw]
excerpt: "여러 AI 에이전트를 운영할 때 가장 골치 아픈 건 '누가 죽었는지 모른다'는 것이다. 에이전트 플릿 헬스 모니터링을 자동화한 실전 경험을 공유한다."
toc: true
---

## 문제: 에이전트가 조용히 죽는다

AI 에이전트를 5대 이상 운영하면 공통적으로 겪는 문제가 있다. **에이전트가 에러 없이 조용히 멈춘다.** OAuth 토큰 만료, 외부 API rate limit, 메모리 누수 등 원인은 다양하다. 사용자가 "왜 답이 없어요?"라고 물어볼 때쯤이면 이미 수 시간째 다운된 상태다.

## 설계: 3단계 헬스 체크

우리 팀이 구축한 모니터링은 크게 3단계로 동작한다.

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│  Heartbeat   │───▶│  Status 집계  │───▶│ Alert/복구   │
│  (per agent) │    │  (cron 기반)  │    │ (자동/수동)  │
└─────────────┘    └──────────────┘    └─────────────┘
```

### 1단계: Heartbeat 수집

각 에이전트는 정해진 시간에 heartbeat를 보낸다. 핵심은 **"응답이 없는 것 자체가 신호"**라는 점이다.

```javascript
// 에이전트별 마지막 활동 시각 추적
const THRESHOLD_HOURS = 24;

function checkAgentHealth(agents) {
  const now = Date.now();
  return agents.map(agent => {
    const lastSeen = agent.lastActivityAt;
    const silentHours = (now - lastSeen) / (1000 * 60 * 60);
    
    return {
      name: agent.name,
      status: silentHours < 6 ? '✅' 
            : silentHours < THRESHOLD_HOURS ? '⚠️' 
            : '🔴',
      silentHours: Math.round(silentHours)
    };
  });
}
```

### 2단계: 상태 집계 리포트

크론(cron)으로 매일 오전 9시에 전체 상태를 집계한다. 출력 형태는 이런 식이다:

```
✅ Agent-A: 정상 (2시간 전 응답)
⚠️ Agent-B: 주의 (18시간 공백)
🔴 Agent-C: 비활성 (3일 공백)
```

단순히 "살아있다/죽었다"가 아니라, **공백 시간에 따른 단계별 분류**가 핵심이다. 6시간 미만은 정상, 6~24시간은 경고, 24시간 초과는 장애로 본다.

### 3단계: 자동 복구 트리거

가장 흔한 장애 원인은 **인증 토큰 만료**다. 이 경우 자동 재인증 플로우를 돌린다.

```bash
#!/bin/bash
# 에이전트 인증 상태 체크 + 재인증
check_and_refresh() {
  local agent_name=$1
  
  # 토큰 유효성 체크
  if ! validate_token "$agent_name"; then
    echo "[$agent_name] 토큰 만료 감지 → 재인증 시도"
    refresh_oauth_token "$agent_name"
    
    # 재인증 후 재확인
    if validate_token "$agent_name"; then
      echo "[$agent_name] ✅ 복구 완료"
      notify_admin "$agent_name 자동 복구 성공"
    else
      echo "[$agent_name] ❌ 자동 복구 실패 → 수동 개입 필요"
      notify_admin "$agent_name 수동 개입 필요" --priority high
    fi
  fi
}
```

## 실전 교훈

일주일간 운영하며 얻은 교훈 3가지:

**1. "공백 = 장애"가 아니다.** 사용자(고객)가 며칠간 메시지를 안 보내면 에이전트도 자연히 조용하다. 단순 공백과 실제 장애를 구분하려면, 사용자의 마지막 메시지 시각도 함께 봐야 한다.

**2. 재인증 자동화가 ROI 최고.** 장애의 60% 이상이 OAuth 토큰 만료였다. 이걸 자동화하니 수동 개입이 확 줄었다.

**3. 리포트는 사람이 읽는 것이다.** 이모지 상태 표시(✅⚠️🔴)가 별거 아닌 것 같지만, 한눈에 파악되는 포맷이 운영 효율을 크게 올린다.

## 마무리

AI 에이전트를 "배포"하는 건 시작일 뿐이다. 진짜 어려운 건 **여러 에이전트를 안정적으로 운영하는 것**이다. 헬스 모니터링은 그 첫 번째 단추다. 작게 시작해서 반복적으로 개선하면 된다.

---

*이 글은 실제 멀티 에이전트 SaaS를 운영하며 얻은 경험을 정리한 것입니다.*
