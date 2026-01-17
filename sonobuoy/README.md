# Sonobuoy

쿠버네티스 클러스터의 **기본 자격(Conformance) 및 클러스터 상태 진단**을 위한 도구.

## 📚 문서 구조

### 기본 문서
- [01-개요.md](./01-개요.md) - Sonobuoy란 무엇인가, 주요 기능 및 특징
- [02-구성요소.md](./02-구성요소.md) - 아키텍처 및 구성 요소 설명
- [03-실행모드.md](./03-실행모드.md) - 실행 모드 종류 및 선택 가이드

### 사용 가이드
- [04-기본사용법.md](./04-기본사용법.md) - 설치부터 실행, 결과 확인까지 기본 워크플로우
- [05-커스터마이징.md](./05-커스터마이징.md) - 플러그인, 필터링, 고급 옵션

### CAT 활용
- [06-CAT-활용.md](./06-CAT-활용.md) - 클러스터 인수테스트(CAT) 관점에서의 Sonobuoy 활용
- [07-검증질문-합격기준.md](./07-검증질문-합격기준.md) - Assertion 기반 검증 질문 및 합격 기준 정의

### 실무 가이드
- [08-실무-체크리스트.md](./08-실무-체크리스트.md) - 실무 적용 시 확인 사항
- [09-CICD-통합.md](./09-CICD-통합.md) - CI/CD 파이프라인 통합 가이드

## 🎯 빠른 시작

```bash
# 1. Sonobuoy 설치 (macOS)
brew install sonobuoy

# 2. 기본 Conformance 테스트 실행
sonobuoy run --mode non-disruptive-conformance --wait

# 3. 결과 확인
results=$(sonobuoy retrieve)
sonobuoy results $results

# 4. 정리
sonobuoy delete --wait
```

## 🔗 관련 리소스

- 공식 사이트: https://sonobuoy.io
- GitHub: https://github.com/vmware-tanzu/sonobuoy
- 문서: https://sonobuoy.io/docs/

## 📋 CAT에서의 역할

Sonobuoy는 **쿠버네티스 클러스터 인수테스트(CAT)**의 다음 카테고리에 해당합니다:

| 카테고리 | 검증 질문(Assertion) | 도구 |
|---------|---------------------|------|
| **기본 자격/기능(Conformance/Config)** | "클러스터가 공식 Kubernetes Conformance를 만족하는가?" | Sonobuoy |

자세한 내용은 [06-CAT-활용.md](./06-CAT-활용.md) 참조.
