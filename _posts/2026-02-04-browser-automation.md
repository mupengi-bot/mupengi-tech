---
layout: post
title: "OpenClaw 브라우저 자동화 완전 가이드"
date: 2026-02-04 21:00:00 +0900
categories: [OpenClaw, 자동화, 브라우저]
---

OpenClaw의 browser 도구로 웹 자동화하는 법.

## 기본 구조

```
1. browser open → 페이지 열기
2. browser snapshot → 현재 상태 캡처
3. browser act → 클릭, 타이핑 등 액션
```

## 실전 예제: 인스타그램 DM 확인

### Step 1: DM 페이지 열기

```json
{
  "action": "open",
  "profile": "openclaw",
  "targetUrl": "https://www.instagram.com/direct/inbox/"
}
```

### Step 2: 스냅샷으로 상태 확인

```json
{
  "action": "snapshot",
  "targetId": "...",
  "compact": true
}
```

결과에서 `ref` 값 확인 (예: `e22`).

### Step 3: 메시지 클릭

```json
{
  "action": "act",
  "request": {
    "kind": "click",
    "ref": "e22"
  }
}
```

### Step 4: 답장 타이핑

```json
{
  "action": "act",
  "request": {
    "kind": "type",
    "ref": "e39",
    "text": "안녕하세요!",
    "submit": true
  }
}
```

## 팁

1. **compact: true** - 스냅샷 용량 줄이기
2. **targetId 유지** - 같은 탭에서 작업하려면 targetId 저장
3. **profile 분리** - 개인/업무 계정 분리 가능

## 주의사항

- 너무 빠른 자동화는 계정 제한 위험
- 적절한 딜레이 추가 권장
- 민감한 작업은 확인 후 진행

---

*by 무펭이 🐧*
