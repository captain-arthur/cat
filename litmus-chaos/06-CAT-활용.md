# CAT 관점에서의 LitmusChaos 활용

LitmusChaos는 **쿠버네티스 클러스터 인수테스트(CAT: Cluster Acceptance Test)**의 **"복원력/장애 대응(Resilience/Fault Tolerance)"** 카테고리에 해당하는 핵심 도구입니다.

## CAT에서의 LitmusChaos 역할

### 카테고리 위치

| 카테고리 | 검증 질문(Assertion) | 도구 | LitmusChaos 역할 |
|---------|---------------------|------|----------------|
| **복원력/장애 대응(Resilience/Fault Tolerance)** | "클러스터가 장애 상황에서도 정상 동작을 유지하는가?" | LitmusChaos | ✅ 핵심 도구 |

### CAT 흐름에서의 위치

```
CAT 단계별 흐름:
1. 기본 적합성(Conformance) → Sonobuoy
2. 보안 기본선(Security baseline) → kube-bench
3. 성능/수용량(Scale) → kube-burner / ClusterLoader2
4. 복원력(Resilience) → LitmusChaos ✅ ← 여기
5. 리포트/자동화 → CI 파이프라인
```

## 검증 질문(Assertion) 중심 접근

### 핵심 원칙

**"도구"가 아니라 "검증 질문(Assertion)"으로 카테고리를 고정**

- 도구는 경계가 모호하다
- 같은 도구라도 질문이 다르면 다른 카테고리에 넣어도 된다
- 필요하다면 "Primary 카테고리 1개 + Secondary 태그"로 운영

### LitmusChaos를 사용하는 검증 질문(Assertion)

| 검증 질문(Assertion) | 테스트 방법 | 합격 기준 예시 |
|---------------------|------------|---------------|
| **"Pod가 죽었을 때 자동으로 복구되는가?"** | Pod Delete 실험 → ChaosResult 확인 | ChaosResult verdict: Pass |
| **"네트워크 지연 상황에서도 서비스가 정상 응답하는가?"** | Network Chaos 실험 → Probe 검증 | Probe 검증 통과 |
| **"리소스 부하 상황에서도 서비스가 정상 동작하는가?"** | CPU/Memory 스트레스 실험 → Probe 검증 | Probe 검증 통과 |

## CAT 기준 정의 예시

### 기준(Policy) + 절차(Procedure) + 증거(Report)

#### 1. 기준(Policy)

| 검증 질문 | 합격 기준 | 비고 |
|----------|----------|------|
| **"Pod 삭제 후 자동 복구가 요구 시간 내에 완료되는가?"** | Pod 삭제 후 60초 이내에 "Running" 상태로 복구 | 고가용성 확인 |
| **"Probe 검증이 통과하는가?"** | Probe 검증 통과 (verdict: Pass) | Steady State 유지 확인 |

#### 2. 절차(Procedure)

```bash
# CAT LitmusChaos 실행 절차 (권장)

# 1단계: ChaosEngine 생성
kubectl apply -f pod-delete.yaml

# 2단계: 실험 상태 확인
kubectl get chaosengine pod-delete-example -n default

# 3단계: ChaosResult 확인
kubectl get chaosresult -n default

# 4단계: Gate 확인
# → ChaosResult verdict 확인
VERDICT=$(kubectl get chaosresult pod-delete-example-pod-delete -n default -o jsonpath='{.status.experimentStatus.verdict}')
if [ "$VERDICT" != "Pass" ]; then
  echo "❌ 복원력 검증 실패: 배포 차단"
  exit 1
fi

# 5단계: 리포트 저장
mkdir -p artifacts
kubectl get chaosresult -n default -o yaml > artifacts/chaos-result-$(date +%Y%m%d).yaml

# 6단계: 정리
kubectl delete chaosengine pod-delete-example -n default
```

#### 3. 증거(Report)

| 산출물 | 설명 | 형태 |
|--------|------|------|
| **ChaosResult CRD** | 실험 실행 결과 원본 | `chaosresult` CRD |
| **ChaosEngine CRD** | 실험 정의 및 상태 | `chaosengine` CRD |
| **Probe 결과** | Probe 검증 결과 | ChaosResult 내 Probe 결과 |

## 관련 문서

- [07-검증질문-합격기준.md](./07-검증질문-합격기준.md) - 검증 질문 및 합격 기준 상세 정의
- [09-CICD-통합.md](./09-CICD-통합.md) - CI/CD 파이프라인 통합 가이드
