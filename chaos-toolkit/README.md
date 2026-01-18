# Chaos Toolkit

쿠버네티스 클러스터의 **복원력(Resilience) 및 장애 대응 능력**을 테스트하는 도구.

## 📚 문서 구조

### 기본 문서
- [01-개요.md](./01-개요.md) - Chaos Toolkit이란 무엇인가, 주요 기능 및 특징
- [02-구성요소.md](./02-구성요소.md) - 아키텍처 및 구성 요소 설명
- [03-설치및사용법.md](./03-설치및사용법.md) - 설치부터 실행, 결과 확인까지 기본 워크플로우

### 실험 가이드
- [04-실험-정의.md](./04-실험-정의.md) - Experiment 파일 작성 및 Steady-State Hypothesis 정의
- [05-Kubernetes-통합.md](./05-Kubernetes-통합.md) - Kubernetes Operator 및 kubectl 확장

### CAT 활용
- [06-CAT-활용.md](./06-CAT-활용.md) - 클러스터 인수테스트(CAT) 관점에서의 Chaos Toolkit 활용
- [07-검증질문-합격기준.md](./07-검증질문-합격기준.md) - Assertion 기반 검증 질문 및 합격 기준 정의

### 실무 가이드
- [08-실무-체크리스트.md](./08-실무-체크리스트.md) - 실무 적용 시 확인 사항
- [09-CICD-통합.md](./09-CICD-통합.md) - CI/CD 파이프라인 통합 가이드

## 🎯 빠른 시작

```bash
# 1. Chaos Toolkit 설치
pip install -U chaostoolkit chaostoolkit-kubernetes

# 2. 실험 파일 작성
chaos init kubernetes

# 3. 실험 실행
chaos run experiment.json

# 4. 리포트 생성
chaos report journal.json report.html
```

## 🔗 관련 리소스

- 공식 사이트: https://chaostoolkit.org
- GitHub: https://github.com/chaostoolkit/chaostoolkit
- 문서: https://chaostoolkit.org/reference/

## 📋 CAT에서의 역할

Chaos Toolkit은 **쿠버네티스 클러스터 인수테스트(CAT)**의 다음 카테고리에 해당합니다:

| 카테고리 | 검증 질문(Assertion) | 도구 |
|---------|---------------------|------|
| **복원력/장애 대응(Resilience/Fault Tolerance)** | "클러스터가 장애 상황에서도 정상 동작을 유지하는가?" | Chaos Toolkit |

자세한 내용은 [06-CAT-활용.md](./06-CAT-활용.md) 참조.
