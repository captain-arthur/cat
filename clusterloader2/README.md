# ClusterLoader2

쿠버네티스 클러스터의 **성능 및 스케일 테스트**를 수행하는 공식 프레임워크.

## 📚 문서 구조

### 기본 문서
- [01-개요.md](./01-개요.md) - ClusterLoader2란 무엇인가, 주요 기능 및 특징
- [02-구성요소.md](./02-구성요소.md) - 아키텍처 및 구성 요소 설명
- [03-설치및사용법.md](./03-설치및사용법.md) - 설치부터 실행, 결과 확인까지 기본 워크플로우
- [04-워크로드-시나리오.md](./04-워크로드-시나리오.md) - 제공되는 워크로드 시나리오 설명

### CAT 활용
- [05-CAT-활용.md](./05-CAT-활용.md) - 클러스터 인수테스트(CAT) 관점에서의 ClusterLoader2 활용
- [06-검증질문-합격기준.md](./06-검증질문-합격기준.md) - Assertion 기반 검증 질문 및 합격 기준 정의

### kube-burner 비교
- [07-kube-burner-비교.md](./07-kube-burner-비교.md) - kube-burner와의 차이점 및 선택 가이드

## 🎯 빠른 시작

```bash
# 1. ClusterLoader2 빌드/설치
git clone https://github.com/kubernetes/perf-tests.git
cd perf-tests/clusterloader2
go build -o clusterloader2 ./cmd/

# 2. 기본 테스트 실행
./clusterloader2 --kubeconfig=$HOME/.kube/config \
  --testconfig=testing/load/config.yaml \
  --report-dir=./reports

# 3. 결과 확인
ls -la reports/
```

## 🔗 관련 리소스

- GitHub: https://github.com/kubernetes/perf-tests/tree/master/clusterloader2
- 공식 문서: Kubernetes perf-tests의 공식 성능 테스트 프레임워크
- 위치: `kubernetes/perf-tests` (Kubernetes 공식 리포지토리)

## 📋 CAT에서의 역할

ClusterLoader2는 **쿠버네티스 클러스터 인수테스트(CAT)**의 다음 카테고리에 해당합니다:

| 카테고리 | 검증 질문(Assertion) | 도구 |
|---------|---------------------|------|
| **성능/수용량(Performance/Scale)** | "클러스터가 요구 성능 수준을 만족하는가?" | ClusterLoader2 |

## 주요 특징

| 특징 | 설명 |
|------|------|
| **공식 프레임워크** | Kubernetes 공식 리포지토리(perf-tests)의 성능 테스트 프레임워크 |
| **내장 Measurement** | PodStartupLatency, APIAvailability, APIResponsiveness 등 내장 측정 기능 |
| **YAML 기반 테스트** | 테스트 시나리오를 YAML로 선언적 정의 |
| **Threshold 검증** | 측정 결과가 threshold를 벗어나면 테스트 실패 처리 가능 |
| **Prometheus 통합** | Prometheus 스택 자동 설치/정리 및 메트릭 수집 |

## kube-burner와의 차이점

| 항목 | ClusterLoader2 | kube-burner |
|------|---------------|-------------|
| **위치** | Kubernetes 공식 리포지토리 (perf-tests) | CNCF Sandbox 프로젝트 |
| **내장 Measurement** | 강함 (PodStartupLatency, APIAvailability 등) | 약함 (Prometheus 의존) |
| **리포팅** | 내장 summary 출력 (p50/p90/p99) | Prometheus 중심 |
| **Prometheus 의존도** | 낮음 (내장 measurement로 시작 가능) | 높음 (메트릭 수집에 필요) |
| **공식성** | 높음 (Kubernetes 공식) | 보통 (실무 도구) |

자세한 내용은 [07-kube-burner-비교.md](./07-kube-burner-비교.md) 참조.
