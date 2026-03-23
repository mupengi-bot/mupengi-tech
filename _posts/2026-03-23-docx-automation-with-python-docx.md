---
title: "python-docx로 정부지원사업 양식 자동화하기"
date: 2026-03-23
categories: [자동화]
tags: [python-docx, docx, 자동화, AI, 사업계획서, 정부지원사업]
excerpt: "원본 양식을 살리면서 AI가 내용을 채우는 DOCX 자동화 파이프라인 구축기"
toc: true
---

## 문제: 양식 무시하면 탈락한다

정부지원사업에 지원할 때 가장 흔한 실수가 뭘까? **원본 양식을 무시하는 것**이다.

내용이 아무리 좋아도 지정된 `.docx` 양식 대신 자체 디자인으로 제출하면 형식 미준수로 탈락한다. 실제로 이런 이유로 탈락한 경험이 있다. 그래서 **원본 양식 위에 AI가 내용을 채우는 자동화 파이프라인**을 만들었다.

## 핵심 아이디어

1. 주최측이 배포한 `.docx` 양식을 그대로 로드
2. 각 섹션(표, 단락)을 탐색하며 빈칸 식별
3. AI가 생성한 콘텐츠를 해당 위치에 삽입
4. 원본 서식(폰트, 표 스타일, 페이지 설정) 100% 유지

## python-docx 기본 구조

```python
from docx import Document

doc = Document("원본_양식.docx")

# 테이블 탐색 — 정부 양식은 대부분 표 기반
for table in doc.tables:
    for row in table.rows:
        for cell in row.cells:
            if is_fillable(cell.text):
                fill_cell(cell, generated_content)

doc.save("완성본.docx")
```

핵심은 `cell.paragraphs`에 접근해서 **기존 Run의 서식을 복사**하는 것이다.

## 서식 보존이 까다로운 이유

`cell.text = "새 내용"`으로 넣으면 서식이 날아간다. 원본 폰트, 크기, 볼드 설정을 유지하려면 Run 단위로 조작해야 한다:

```python
def fill_cell(cell, content):
    """기존 서식을 보존하면서 내용 교체"""
    para = cell.paragraphs[0]
    if para.runs:
        # 첫 번째 Run의 서식 복사
        template_run = para.runs[0]
        font_name = template_run.font.name
        font_size = template_run.font.size
        is_bold = template_run.font.bold

        # 기존 Run 모두 삭제
        for run in para.runs:
            run.text = ""
        
        # 새 내용을 첫 Run에 삽입 (서식 유지)
        para.runs[0].text = content
    else:
        para.add_run(content)
```

## 표 안의 표 (Nested Table) 처리

정부 양식은 표 안에 표가 들어있는 경우가 많다. `python-docx`는 nested table을 직접 지원하지 않지만, XML 레벨에서 접근하면 된다:

```python
from docx.oxml.ns import qn

def get_nested_tables(cell):
    """셀 안의 중첩 테이블 추출"""
    nested = []
    for tbl in cell._element.findall(qn('w:tbl')):
        nested.append(Table(tbl, cell._element))
    return nested
```

## AI 콘텐츠 생성 + 삽입 파이프라인

전체 흐름은 이렇다:

```
원본 양식 로드 → 섹션 매핑 → AI 콘텐츠 생성 → 셀별 삽입 → 검증 → 저장
```

섹션 매핑은 양식마다 다르기 때문에, 첫 번째 패스에서 모든 셀의 텍스트를 추출하고 어떤 셀에 어떤 내용을 넣을지 매핑 테이블을 만든다.

```python
section_map = {
    "아이템명": "AI 에이전트 관리형 서비스",
    "창업 동기": ai_generate("창업 동기 300자"),
    "시장 분석": ai_generate("TAM/SAM/SOM 분석"),
}
```

## 배운 것

1. **양식은 신성하다** — 절대 자체 디자인으로 대체하지 말 것
2. **Run 단위 조작** — `cell.text` 직접 할당은 서식 파괴의 지름길
3. **버전 관리 필수** — v1~v7까지 반복 수정이 불가피하므로 각 버전을 파일명으로 관리
4. **검증 단계** — 저장 후 Word로 열어서 레이아웃 깨짐이 없는지 반드시 확인

정부지원사업 시즌에 `.docx` 양식 작성을 자동화하면, 내용에만 집중할 수 있다. 양식 걱정은 코드에 맡기자.
