# CAT 관점에서의 Chaos Mesh 활용

Chaos Mesh는 **쿠버네티스 클러스터 인수테스트(CAT: Cluster Acceptance Test)**의 **"복원력/장애 대응(Resilience/Fault Tolerance)"** 카테고리에 해당하는 핵심 도구입니다.

## CAT에서의 Chaos Mesh 역할

### 카테고리 위치

| 카테고리 | 검증 질문(Assertion) | 도구 | Chaos Mesh 역할 |
|---------|---------------------|------|---------------|
| **복원력/장애 대응(Resilience/Fault Tolerance)** | "클러스터가 장애 상황에서도 정상 동작을 유지하는가?" | Chaos Mesh | ✅ 핵심 도구 |

### CAT 흐름에서의 위치

```
CAT 단계별 흐름:
1. 기본 적합성(Conformance) → Sonobuoy
2. 보안 기본선(Security baseline) → kube-bench
3. 성능/수용량(Scale) → kube-burner / ClusterLoader2
4. 복원력(Resilience) → Chaos Mesh ✅ ← 여기
5. 리포트/자동화 → CI 파이프라인
```

## 검증 질문(Assertion) 중심 접근

### 핵심 원칙

**"도구"가 아니라 "검증 질문(Assertion)"으로 카테고리를 고정**

- 도구는 경계가 모호하다
- 같은 도구라도 질문이 다르면 다른 카테고리에 넣어도 된다
- 필요하다면 "Primary 카테고리 1개 + Secondary 태그"로 운영

### Chaos Mesh를 사용하는 검증 질문(Assertion)

| 검증 질문(Assertion) | 테스트 방법 | 합격 기준 예시 |
|---------------------|------------|---------------|
| **"Pod가 죽었을 때 자동으로 복구되는가?"** | PodChaos (pod-kill) → Pod 상태 확인 | 실험 후 Pod 상태가 "Running"으로 복구 |
| **"네트워크 지연 상황에서도 서비스가 정상 응답하는가?"** | NetworkChaos (delay) → HTTP 응답 확인 | 서비스가 정상 응답 유지 |
| **"리소스 부하 상황에서도 서비스가 정상 동작하는가?"** | StressChaos (CPU/Memory) → 서비스 응답 확인 | 서비스가 정상 응답 유지 |

## 리스크 중심 접근

### 리스크와 Chaos Mesh 검증

| 리스크 | 장애 징후 | 테스트 | Chaos Mesh 역할 | 합격 기준 |
|--------|----------|--------|---------------|----------|
| **Pod 장애** | Pod가 죽었을 때 서비스 중단 | PodChaos (pod-kill) → 자동 복구 확인 | PodChaos CRD 실행 | Pod 상태가 "Running"으로 복구 |
| **네트워크 지연** | 네트워크 지연 시 서비스 응답 저하 | NetworkChaos (delay) → 응답 시간 확인 | NetworkChaos CRD 실행 | 응답 시간이 허용 범위 내 |
| **리소스 부하** | CPU/메모리 부하 시 성능 저하 | StressChaos → 서비스 응답 확인 | StressChaos CRD 실행 | 서비스가 정상 응답 유지 |

## CAT 기준 정의 예시

### 기준(Policy) + 절차(Procedure) + 증거(Report)

#### 1. 기준(Policy)

| 검증 질문 | 합격 기준 | 비고 |
|----------|----------|------|
| **"Pod 삭제 후 자동 복구가 요구 시간 내에 완료되는가?"** | Pod 삭제 후 60초 이내에 "Running" 상태로 복구 | 고가용성 확인 |
| **"네트워크 지연 상황에서도 서비스가 정상 응답하는가?"** | 네트워크 지연 주입 후 HTTP 200 응답 유지 | 네트워크 복원력 확인 |
| **"리소스 부하 상황에서도 서비스가 정상 동작하는가?"** | CPU/메모리 부하 주입 후 서비스가 정상 응답 유지 | 리소스 복원력 확인 |

#### 2. 절차(Procedure)

```bash
# CAT Chaos Mesh 실행 절차 (권장)

# 1단계: 실험 CRD 생성
kubectl apply -f pod-kill.yaml

# 2단계: 실험 상태 확인
kubectl get podchaos pod-kill-example -n default

# 3단계: 실험 실행 중 Pod 상태 확인
kubectl get pods -l app=test-app -n default -w

# 4단계: 실험 종료 후 상태 확인
kubectl get podchaos pod-kill-example -n default -o jsonpath='{.status}'

# 5단계: Gate 확인
# → 실험 실패 시 배포 차단
STATUS=$(kubectl get podchaos pod-kill-example -n default -o jsonpath='{.status.experiment.phase}')
if [ "$STATUS" != "Running" ] && [ "$STATUS" != "Finished" ]; then
  echo "❌ 복원력 검증 실패: 배포 차단"
  exit 1
fi

# 6단계: 리포트 저장
mkdir -p artifacts
kubectl get podchaos pod-kill-example -n default -o yaml > artifacts/pod-kill-$(date +%Y%m%d).yaml

# 7단계: 정리 (자동 복구)
kubectl delete podchaos pod-kill-example -n default
```

#### 3. 증거(Report)

| 산출물 | 설명 | 형태 |
|--------|------|------|
| **CRD YAML** | Chaos Experiment CRD 정의 및 상태 | `.yaml` 파일 |
| **Status 필드** | 실험 실행 결과 상태 | CRD Status 필드 |
| **이벤트** | Kubernetes 이벤트 (kubectl describe) | kubectl 출력 |

## 실무 적용 시나리오

### 시나리오 1: Pod 자동 복구 테스트

**목적:** Pod 삭제 시 자동으로 복구되는지 확인

**절차:**

```bash
# 1. 실험 CRD 생성
kubectl apply -f pod-kill.yaml

# 2. 실험 상태 확인
kubectl get podchaos pod-kill-test -n default

# 3. Pod 상태 확인
kubectl get pods -l app=test-app -n default

# 4. Gate 확인
STATUS=$(kubectl get podchaos pod-kill-test -n default -o jsonpath='{.status.experiment.phase}')
if [ "$STATUS" != "Running" ] && [ "$STATUS" != "Finished" ]; then
  echo "❌ Pod 자동 복구 검증 실패"
  exit 1
fi

# 5. 정리 (자동 복구)
kubectl delete podchaos pod-kill-test -n default
```

**합격 기준:** 실험 종료 후 Pod 상태가 "Running"으로 복구

### 시나리오 2: 네트워크 지연 복원력 테스트

**목적:** 네트워크 지연 상황에서도 서비스가 정상 응답하는지 확인

**절차:**

```bash
# 1. 실험 CRD 생성
kubectl apply -f network-delay.yaml

# 2. 실험 상태 확인
kubectl get networkchaos network-delay-test -n default

# 3. 서비스 응답 확인 (별도 도구 사용)
# curl 또는 HTTP 테스트 도구로 응답 확인

# 4. Gate 확인
STATUS=$(kubectl get networkchaos network-delay-test -n default -o jsonpath='{.status.experiment.phase}')
if [ "$STATUS" != "Running" ] && [ "$STATUS" != "Finished" ]; then
  echo "❌ 네트워크 복원력 검증 실패"
  exit 1
fi

# 5. 정리 (자동 복구)
kubectl delete networkchaos network-delay-test -n default
```

**합격 기준:** 네트워크 지연 주입 후 서비스가 정상 응답 유지

## 자동화 통합

### CI/CD 파이프라인 예시

```yaml
# .github/workflows/cat-chaos-mesh.yml (GitHub Actions 예시)
name: CAT - Chaos Mesh Resilience Test

on:
  workflow_dispatch:
  schedule:
    - cron: '0 3 * * 0'  # 주 1회 (일요일 새벽 3시)

jobs:
  chaos-resilience:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Configure kubectl
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG }}
        run: |
          echo "$KUBECONFIG" | base64 -d > ~/.kube/config

      - name: Run Experiment
        run: |
          kubectl apply -f chaos-mesh/pod-kill.yaml

      - name: Wait for Experiment
        run: |
          kubectl wait --for=condition=Ready podchaos/pod-kill-example -n default --timeout=5m

      - name: Check Results
        run: |
          STATUS=$(kubectl get podchaos pod-kill-example -n default -o jsonpath='{.status.experiment.phase}')
          if [ "$STATUS" != "Running" ] && [ "$STATUS" != "Finished" ]; then
            echo "❌ 복원력 검증 실패"
            exit 1
          fi
          echo "✅ 복원력 검증 성공"

      - name: Cleanup
        if: always()
        run: |
          kubectl delete podchaos pod-kill-example -n default
```

## 관련 문서

- [07-검증질문-합격기준.md](./07-검증질문-합격기준.md) - 검증 질문 및 합격 기준 상세 정의
- [09-CICD-통합.md](./09-CICD-통합.md) - CI/CD 파이프라인 통합 가이드
