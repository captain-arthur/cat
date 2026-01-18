# kube-burner

쿠버네티스 클러스터의 **성능 및 스케일 테스트**를 수행하는 도구.

## 📚 문서 구조

### 기본 문서
- [01-개요.md](./01-개요.md) - kube-burner란 무엇인가, 주요 기능 및 특징
- [02-구성요소.md](./02-구성요소.md) - 아키텍처 및 구성 요소 설명
- [03-설치및사용법.md](./03-설치및사용법.md) - 설치부터 실행, 결과 확인까지 기본 워크플로우
- [04-워크로드-시나리오.md](./04-워크로드-시나리오.md) - 제공되는 워크로드 시나리오 설명

### CAT 활용
- [05-CAT-활용.md](./05-CAT-활용.md) - 클러스터 인수테스트(CAT) 관점에서의 kube-burner 활용
- [06-검증질문-합격기준.md](./06-검증질문-합격기준.md) - Assertion 기반 검증 질문 및 합격 기준 정의

### 실무 가이드
- [07-실무-체크리스트.md](./07-실무-체크리스트.md) - 실무 적용 시 확인 사항
- [08-CICD-통합.md](./08-CICD-통합.md) - CI/CD 파이프라인 통합 가이드

## 🎯 빠른 시작

```bash
# 1. kube-burner 설치
go install github.com/cloud-bulldozer/kube-burner@latest

# 또는 바이너리 다운로드
# https://github.com/cloud-bulldozer/kube-burner/releases

# 2. 기본 워크로드 실행
kube-burner init -c cluster-density.yaml

# 3. 결과 확인
ls -la reports/
```

## 🔗 관련 리소스

- GitHub: https://github.com/cloud-bulldozer/kube-burner
- 문서: https://kube-burner.github.io/kube-burner/
- CNCF Sandbox 프로젝트

## 📋 CAT에서의 역할

kube-burner는 **쿠버네티스 클러스터 인수테스트(CAT)**의 다음 카테고리에 해당합니다:

| 카테고리 | 검증 질문(Assertion) | 도구 |
|---------|---------------------|------|
| **성능/수용량(Performance/Scale)** | "클러스터가 요구 성능 수준을 만족하는가?" | kube-burner |

## 주요 특징

| 특징 | 설명 |
|------|------|
| **대량 리소스 생성** | 대규모 Pod/Namespace/Deployment 등을 생성하여 부하 테스트 |
| **성능 지표 수집** | Prometheus와 연계하여 Pod latency, API latency 등 측정 |
| **다양한 워크로드** | cluster-density, node-density, networkpolicy 등 시나리오 제공 |
| **임계치 검증** | p50, p95, p99 등 통계 및 threshold 기반 Pass/Fail 판단 |

## Sonobuoy와의 차이점

| 항목 | Sonobuoy | kube-burner |
|------|----------|-------------|
| **목적** | Conformance 테스트 (기본 자격 검증) | 성능/스케일 테스트 |
| **검증 축** | 기능 정상 동작 여부 | 성능 수준 및 수용량 |
| **측정** | Pass/Fail (기능 테스트) | Latency, Throughput (성능 메트릭) |

## 주요 워크로드 시나리오

| 워크로드 | 목적 |
|---------|------|
| **cluster-density** | 컨트롤 플레인 성능 테스트 (다양한 리소스 밀도) |
| **node-density** | 노드별 파드 밀도 테스트 |
| **networkpolicy** | 네트워크 정책 오버헤드 테스트 |
| **crd-scale** | CRD 스케일 성능 테스트 |

자세한 내용은 [04-워크로드-시나리오.md](./04-워크로드-시나리오.md) 참조.
