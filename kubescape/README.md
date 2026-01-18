# Kubescape

쿠버네티스 클러스터의 **보안 설정 및 취약점**을 검사하는 Kubernetes 보안 스캔 도구.

## 📚 문서 구조

### 기본 문서
- [01-개요.md](./01-개요.md) - Kubescape란 무엇인가, 주요 기능 및 특징
- [02-구성요소.md](./02-구성요소.md) - 아키텍처 및 구성 요소 설명
- [03-설치및사용법.md](./03-설치및사용법.md) - 설치부터 실행, 결과 확인까지 기본 워크플로우

### 스캔 가이드
- [04-스캔-방법.md](./04-스캔-방법.md) - 다양한 스캔 방법 및 옵션
- [05-프레임워크-컨트롤.md](./05-프레임워크-컨트롤.md) - 보안 프레임워크 및 컨트롤 정의

### CAT 활용
- [06-CAT-활용.md](./06-CAT-활용.md) - 클러스터 인수테스트(CAT) 관점에서의 Kubescape 활용
- [07-검증질문-합격기준.md](./07-검증질문-합격기준.md) - Assertion 기반 검증 질문 및 합격 기준 정의

### 실무 가이드
- [08-실무-체크리스트.md](./08-실무-체크리스트.md) - 실무 적용 시 확인 사항
- [09-CICD-통합.md](./09-CICD-통합.md) - CI/CD 파이프라인 통합 가이드

## 🎯 빠른 시작

```bash
# 1. Kubescape 설치
curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash

# 2. 기본 스캔 (NSA-CISA 프레임워크)
kubescape scan framework nsa

# 3. 클러스터 스캔
kubescape scan --enable-host-scan

# 4. 결과를 JSON으로 저장
kubescape scan framework nsa --format json --output results.json
```

## 🔗 관련 리소스

- 공식 사이트: https://kubescape.io
- GitHub: https://github.com/kubescape/kubescape
- 문서: https://kubescape.io/docs/

## 📋 CAT에서의 역할

Kubescape는 **쿠버네티스 클러스터 인수테스트(CAT)**의 다음 카테고리에 해당합니다:

| 카테고리 | 검증 질문(Assertion) | 도구 |
|---------|---------------------|------|
| **보안 기본선(Security baseline)** | "클러스터 보안 설정이 기준을 만족하는가?" | Kubescape |

자세한 내용은 [06-CAT-활용.md](./06-CAT-활용.md) 참조.
