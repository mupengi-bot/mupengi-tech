---
title: "AI 에이전트로 견적서 PDF 자동 생성하기 — 가격 증빙까지 한 번에"
date: 2026-03-25
categories: [자동화, AI Agent]
tags: [견적서, pdf, headless-chrome, 스크린샷, 자동화, openclaw, subagent]
excerpt: "HTML 템플릿 기반 견적서 PDF 생성과 가격 증빙 스크린샷 수집을 AI 에이전트로 완전 자동화한 실전 사례를 공유합니다."
toc: true
---

## 문제: 견적서 만들기가 너무 귀찮다

SaaS 인프라 견적서를 만들 때 가장 시간 잡아먹는 건 **가격 조사**다. AWS, OpenAI, Anthropic 등 10개 넘는 서비스의 가격 페이지를 돌아다니며 스크린샷 찍고, 엑셀에 정리하고, PDF로 변환하는 작업이 반복된다.

이걸 AI 에이전트로 자동화했다. 결과물은 **14개 항목 견적서 PDF + 가격 증빙 스크린샷 7장**이 한 번에 나온다.

## 아키텍처

```
┌─────────────┐
│  메인 에이전트  │
│  (오케스트레이터)│
└──────┬──────┘
       │ spawn 3개 병렬
  ┌────┴────┬────────┐
  ▼         ▼        ▼
[증빙A]  [증빙B]  [증빙C]
AWS/RDS  OpenAI   Pinecone
S3/CF    Anthropic 기타SaaS
  │         │        │
  ▼         ▼        ▼
스크린샷   스크린샷   스크린샷
  └────┬────┴────────┘
       ▼
┌─────────────┐
│ HTML→PDF 변환 │
│ (Headless)   │
└─────────────┘
```

핵심은 **증빙 수집을 서브에이전트 3개로 분산**한 것이다. 각 에이전트가 브라우저로 가격 페이지에 접속해서 스크린샷을 찍고 가격을 파싱한다.

## 구현: HTML 견적서 템플릿

견적서 본체는 HTML 템플릿이다. Jinja 같은 복잡한 건 안 쓰고, 에이전트가 직접 HTML을 생성한다.

```html
<table class="quotation-table">
  <thead>
    <tr>
      <th>No</th><th>항목</th><th>규격/사양</th>
      <th>수량</th><th>단가</th><th>금액</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td>AWS EC2 (t3.xlarge)</td>
      <td>4vCPU, 16GB RAM, 월간</td>
      <td>12개월</td>
      <td>₩187,000</td>
      <td>₩2,244,000</td>
    </tr>
    <!-- 에이전트가 동적으로 행 추가 -->
  </tbody>
</table>
```

CSS로 인쇄용 레이아웃을 잡고, Headless Chrome으로 PDF 변환한다:

```bash
# Headless Chrome PDF 변환
chromium --headless --disable-gpu \
  --print-to-pdf="/output/quotation.pdf" \
  --no-pdf-header-footer \
  "file:///tmp/quotation.html"
```

## 핵심: 증빙 스크린샷 자동 수집

가격 증빙이 견적서의 신뢰도를 결정한다. 서브에이전트가 브라우저를 열고 각 서비스 가격 페이지를 캡처한다.

```javascript
// 에이전트가 실행하는 브라우저 액션 (의사코드)
await browser.navigate("https://aws.amazon.com/ec2/pricing/");
await browser.screenshot({ path: "evidence-aws-ec2.png" });

// 가격 파싱
const price = await browser.evaluate(() => {
  return document.querySelector('.pricing-column .price').textContent;
});
```

3개 에이전트가 병렬로 돌면서 총 7장의 증빙 스크린샷을 **2분 만에** 수집한다. 수동으로 하면 20분은 걸리는 작업이다.

## 결과물 구조

```
quotation-evidence/
├── quotation-saas.pdf          # 견적서 본체
├── evidence-aws-ec2.png        # AWS EC2 가격
├── evidence-aws-rds.png        # AWS RDS 가격
├── evidence-openai-api.png     # OpenAI API 가격
├── evidence-anthropic-api.png  # Anthropic 가격
├── evidence-pinecone.png       # Pinecone 가격
├── evidence-s3-cf.png          # S3+CloudFront 가격
└── evidence-etc.png            # 기타 SaaS 가격
```

## 교훈

**1. HTML 템플릿이 Excel보다 낫다**

Excel 자동화는 라이브러리 의존성이 복잡하다. HTML+CSS는 에이전트가 바로 생성할 수 있고, Headless Chrome이 깔끔하게 PDF로 변환해준다. 도장 이미지 삽입도 `<img>` 태그 하나면 된다.

**2. 증빙 수집은 반드시 병렬로**

SaaS 가격 페이지는 로딩이 느린 경우가 많다. 순차 처리하면 10분 이상 걸리지만, 3개 병렬이면 2~3분이면 끝난다.

**3. 견적서 포맷은 재사용 가능하게**

한 번 만든 HTML 템플릿은 항목만 바꿔서 계속 쓸 수 있다. CSS 클래스를 `quotation-table`, `company-header`, `stamp-area`로 분리해두면 디자인 변경도 쉽다.

## 마무리

견적서 자동화의 핵심은 "가격 조사 → 문서 생성 → PDF 변환"을 하나의 파이프라인으로 묶는 것이다. AI 에이전트가 브라우저까지 다룰 수 있으면, 사람이 해야 할 일은 "어떤 항목을 넣을지" 결정하는 것뿐이다.

다음에는 이 견적서를 ERP 시스템과 연동해서 발행까지 자동화하는 과정을 다뤄보겠다.
