---
title: "AI 봇끼리 토론시키기 — 턴제 멀티에이전트 채팅 설계"
date: 2026-03-09
categories: [AI Agent]
tags: [multi-agent, orchestration, turn-based, openclaw, discord]
excerpt: "AI 에이전트 여러 대를 한 채팅방에 넣고 토론시키면 어떻게 될까? 실전에서 겪은 문제와 턴제 프로토콜로 해결한 과정을 공유한다."
toc: true
---

## 문제: 봇끼리 대화하면 뭐가 터지나

AI 에이전트 2대 이상을 같은 디스코드 채널에 넣고 주제를 던져봤다. "비타민C, 천연 vs 합성" 같은 가벼운 토론을.

결과는 혼돈이었다:

- **무한 루프**: A가 말하면 B가 반응, B 반응에 A가 또 반응 → 끝없는 핑퐁
- **컨텍스트 꼬임**: 서로의 메시지를 "사용자 입력"으로 인식해서 같은 질문 반복
- **토큰 폭발**: 10분 만에 수만 토큰 소모

멀티에이전트 오케스트레이션에서 가장 기본적인 문제 — **누가 언제 말할 것인가** — 를 풀어야 했다.

## 해결: 턴제(Turn-based) 프로토콜

게임의 턴제 전투에서 아이디어를 빌렸다. 핵심 규칙 3가지:

### 1. 라운드 제한

```yaml
# 최대 라운드 수 (기본값 4)
max_rounds: 4
```

토론 주제당 최대 N라운드. 라운드가 끝나면 강제 종료한다. 무한 루프의 근본 차단.

### 2. 턴 순서 고정

```javascript
// 참가자 배열 순서대로 발언
const participants = ['무펭이', '김대리', '폴'];
let currentTurn = 0;

function nextSpeaker() {
  const speaker = participants[currentTurn % participants.length];
  currentTurn++;
  return speaker;
}
```

자기 턴이 아니면 `NO_REPLY`로 침묵한다. 이게 없으면 누가 먼저 응답하느냐에 따라 대화가 꼬인다.

### 3. 응답 규칙

```markdown
## 에이전트 채팅 규칙
- 자기 턴에만 발언 (아니면 NO_REPLY)
- 1회 발언 = 최대 200자
- 리액션(이모지) 금지 — 트리거 루프 유발
- 마지막 라운드에 결론 한 줄 필수
```

특히 **리액션 금지**가 중요했다. 디스코드에서 이모지 리액션도 이벤트로 잡히기 때문에, 봇이 리액션 → 상대 봇이 반응 → 또 리액션하는 2차 루프가 생긴다.

## 구현 포인트

오케스트레이터(사회자) 역할이 핵심이다:

```python
class TurnOrchestrator:
    def __init__(self, participants, max_rounds=4):
        self.participants = participants
        self.max_rounds = max_rounds
        self.current_round = 0
        self.turn_index = 0

    def should_speak(self, agent_name):
        """이 에이전트가 지금 말할 차례인가?"""
        expected = self.participants[self.turn_index]
        return agent_name == expected

    def advance(self):
        """다음 턴으로"""
        self.turn_index += 1
        if self.turn_index >= len(self.participants):
            self.turn_index = 0
            self.current_round += 1

    def is_finished(self):
        return self.current_round >= self.max_rounds
```

실제 운영에서는 각 에이전트의 시스템 프롬프트에 턴 규칙을 주입하고, 오케스트레이터가 "네 차례야"라는 시그널을 보내는 방식으로 동작한다.

## 실전 결과

디스코드 서버에서 3대의 에이전트로 10라운드 비타민C 토론을 돌렸다. 반응이 꽤 좋았다:

- 각자 다른 관점(과학적 근거 vs 가성비 vs 소비자 경험)으로 논쟁
- 토큰 소모량 예측 가능 (라운드 × 참가자 × 200자)
- 인간이 중간에 끼어들어 질문하는 것도 가능

## 교훈

멀티에이전트 시스템에서 **제약이 곧 품질**이다. 자유롭게 풀어놓으면 혼돈이고, 턴·길이·라운드를 제한하면 오히려 밀도 높은 대화가 나온다.

다음 단계로는 "주제에 따라 전문가 에이전트를 동적으로 소환하는" 시스템을 만들어볼 계획이다.

---

*이 글은 실제 운영 중인 AI 에이전트 시스템에서의 경험을 바탕으로 작성했습니다.*
