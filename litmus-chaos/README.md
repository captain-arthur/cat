# LitmusChaos

쿠버네티스 클러스터의 **복원력(Resilience) 및 장애 대응 능력**을 테스트하는 Kubernetes 네이티브 Chaos Engineering 플랫폼.

## 📚 문서 구조

### 기본 문서
- [01-개요.md](./01-개요.md) - LitmusChaos란 무엇인가, 주요 기능 및 특징
- [02-구성요소.md](./02-구성요소.md) - 아키텍처 및 구성 요소 설명
- [03-설치및사용법.md](./03-설치및사용법.md) - 설치부터 실행, 결과 확인까지 기본 워크플로우

### 실험 가이드
- [04-실험-정의.md](./04-실험-정의.md) - Chaos Experiment 정의 및 실험 유형
- [05-워크플로우.md](./05-워크플로우.md) - Chaos Workflow를 통한 복합 실험 구성

### CAT 활용
- [06-CAT-활용.md](./06-CAT-활용.md) - 클러스터 인수테스트(CAT) 관점에서의 LitmusChaos 활용
- [07-검증질문-합격기준.md](./07-검증질문-합격기준.md) - Assertion 기반 검증 질문 및 합격 기준 정의

### 실무 가이드
- [08-실무-체크리스트.md](./08-실무-체크리스트.md) - 실무 적용 시 확인 사항
- [09-CICD-통합.md](./09-CICD-통합.md) - CI/CD 파이프라인 통합 가이드

## 🎯 빠른 시작

```bash
# 1. LitmusChaos 설치 (Helm)
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm repo update
helm install litmus litmuschaos/litmus --namespace=litmus --create-namespace

# 2. ChaosCenter 접근 (Port Forward)
kubectl port-forward -n litmus svc/litmusportal-frontend 9091:9091

# 3. Chaos Experiment 생성 (예시: Pod Delete)
kubectl apply -f pod-delete.yaml

# 4. 실험 상태 확인
kubectl get chaosengine
kubectl get chaosresult
```

## 🔗 관련 리소스

- 공식 사이트: https://litmuschaos.io
- GitHub: https://github.com/litmuschaos/litmus
- 문서: https://docs.litmuschaos.io

## 📋 CAT에서의 역할

LitmusChaos는 **쿠버네티스 클러스터 인수테스트(CAT)**의 다음 카테고리에 해당합니다:

| 카테고리 | 검증 질문(Assertion) | 도구 |
|---------|---------------------|------|
| **복원력/장애 대응(Resilience/Fault Tolerance)** | "클러스터가 장애 상황에서도 정상 동작을 유지하는가?" | LitmusChaos |

자세한 내용은 [06-CAT-활용.md](./06-CAT-활용.md) 참조.
