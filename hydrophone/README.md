# Hydrophone

쿠버네티스 클러스터의 **기본 자격(Conformance) 테스트**를 실행하기 위한 **경량 러너(lightweight runner)** 도구.

## 📚 문서 구조

### 기본 문서
- [01-개요.md](./01-개요.md) - Hydrophone이란 무엇인가, 주요 기능 및 특징
- [02-구성요소.md](./02-구성요소.md) - 아키텍처 및 구성 요소 설명
- [03-Sonobuoy-비교.md](./03-Sonobuoy-비교.md) - Sonobuoy와의 차이점 및 선택 가이드
- [04-기본사용법.md](./04-기본사용법.md) - 설치부터 실행, 결과 확인까지 기본 워크플로우
- [05-옵션및고급사용.md](./05-옵션및고급사용.md) - 테스트 필터링, 고급 옵션

### CAT 활용
- [06-CAT-활용.md](./06-CAT-활용.md) - 클러스터 인수테스트(CAT) 관점에서의 Hydrophone 활용
- [07-검증질문-합격기준.md](./07-검증질문-합격기준.md) - Assertion 기반 검증 질문 및 합격 기준 정의

### 실무 가이드
- [08-실무-체크리스트.md](./08-실무-체크리스트.md) - 실무 적용 시 확인 사항
- [09-CICD-통합.md](./09-CICD-통합.md) - CI/CD 파이프라인 통합 가이드

## 🎯 빠른 시작

```bash
# 1. Hydrophone 설치 (Go 환경 필요)
go install sigs.k8s.io/hydrophone@latest

# 2. 기본 Conformance 테스트 실행
hydrophone run --wait

# 3. 결과 확인 (e2e.log, junit_01.xml)
# 결과는 현재 디렉토리에 생성됨

# 4. 정리
kubectl delete namespace conformance
```

## 🔗 관련 리소스

- 공식 블로그: https://www.kubernetes.dev/blog/2024/05/23/introducing-hydrophone/
- GitHub: https://github.com/kubernetes-sigs/hydrophone
- Kubernetes Conformance: https://github.com/cncf/k8s-conformance

## 📋 CAT에서의 역할

Hydrophone은 **쿠버네티스 클러스터 인수테스트(CAT)**의 다음 카테고리에 해당합니다:

| 카테고리 | 검증 질문(Assertion) | 도구 |
|---------|---------------------|------|
| **기본 자격/기능(Conformance/Config)** | "클러스터가 공식 Kubernetes Conformance를 만족하는가?" | Hydrophone |

## Sonobuoy와의 차이점

| 항목 | Hydrophone | Sonobuoy |
|------|-----------|----------|
| **목적** | Conformance 테스트 실행에 집중 (경량) | 다양한 플러그인 및 데이터 수집 지원 (풀 기능) |
| **설치** | `go install` (단일 바이너리) | 바이너리 다운로드 또는 패키지 관리자 |
| **실행 방식** | `conformance` 네임스페이스에서 Pod 실행 | `sonobuoy` 네임스페이스에서 Aggregator + 플러그인 실행 |
| **결과 출력** | `e2e.log`, `junit_01.xml` | `results.tar.gz` (다양한 플러그인 결과) |
| **플러그인** | 없음 (단순 실행) | 다양한 플러그인 지원 |

자세한 내용은 [03-Sonobuoy-비교.md](./03-Sonobuoy-비교.md) 참조.
