# CAT 관점에서의 kube-burner 활용

kube-burner는 **쿠버네티스 클러스터 인수테스트(CAT: Cluster Acceptance Test)**의 **"성능/수용량(Performance/Scale)"** 카테고리에 해당하는 핵심 도구입니다.

## CAT에서의 kube-burner 역할

### 카테고리 위치

| 카테고리 | 검증 질문(Assertion) | 도구 | kube-burner 역할 |
|---------|---------------------|------|----------------|
| **성능/수용량(Performance/Scale)** | "클러스터가 요구 성능 수준을 만족하는가?" | kube-burner | ✅ 핵심 도구 |

### CAT 흐름에서의 위치

```
CAT 단계별 흐름:
1. 기본 적합성(Conformance) → Sonobuoy
2. 보안 기본선(Security baseline) → kube-bench
3. 성능/수용량(Scale) → kube-burner ✅ ← 여기
4. 복원력(Resilience) → LitmusChaos
5. 리포트/자동화 → CI 파이프라인
```

## 검증 질문(Assertion) 중심 접근

### 핵심 원칙

**"도구"가 아니라 "검증 질문(Assertion)"으로 카테고리를 고정**

- 도구는 경계가 모호하다
- 같은 도구라도 질문이 다르면 다른 카테고리에 넣어도 된다
- 필요하다면 "Primary 카테고리 1개 + Secondary 태그"로 운영

### kube-burner를 사용하는 검증 질문(Assertion)

| 검증 질문(Assertion) | 테스트 방법 | 합격 기준 예시 |
|---------------------|------------|---------------|
| **"클러스터가 요구 성능 수준을 만족하는가?"** | `kube-burner init -c cluster-density.yml` | p95 API latency < X ms |
| **"N 노드에서 X Pod 생성 시 성능이 기준을 충족하는가?"** | `kube-burner init -c node-density.yml` | Pod Ready 시간 < Y 초 |
| **"컨트롤 플레인 병목 없이 확장 가능한가?"** | `kube-burner init -c cluster-density.yml` | etcd latency < Z ms |
| **"최대 수용량이 요구 수준인가?"** | `kube-burner init -c node-density.yml` | 노드당 최대 N Pod 배포 가능 |

## 리스크 중심 접근

### 리스크와 kube-burner 검증

| 리스크 | 장애 징후 | 테스트 | kube-burner 역할 | 합격 기준 |
|--------|----------|--------|----------------|----------|
| **컨트롤 플레인 병목** | API 느려짐, etcd 지연 | 대량 리소스 생성 | cluster-density 워크로드 | p95 API latency < X ms |
| **노드 수용량 부족** | Pod 스케줄 실패 | 노드별 파드 밀도 테스트 | node-density 워크로드 | 노드당 최대 N Pod |
| **네트워크 정책 오버헤드** | 네트워크 성능 저하 | NetworkPolicy 적용 테스트 | networkpolicy 워크로드 | 통신 지연 증가 < Y% |

## CAT 기준 정의 예시

### 기준(Policy) + 절차(Procedure) + 증거(Report)

#### 1. 기준(Policy)

| 검증 질문 | 합격 기준 | 비고 |
|----------|----------|------|
| **"p95 API latency가 요구 수준인가?"** | p95 API latency < 100ms | 컨트롤 플레인 성능 |
| **"Pod Ready 시간이 요구 수준인가?"** | Pod Ready 시간 < 5초 | 워크로드 배포 성능 |
| **"노드당 최대 파드 수가 요구 수준인가?"** | 노드당 최대 100 Pod 배포 가능 | 수용량 확인 |

#### 2. 절차(Procedure)

```bash
# CAT kube-burner 실행 절차 (권장)

# 1단계: 컨트롤 플레인 성능 테스트
kube-burner init -c cluster-density.yml \
  --prometheus-url http://prometheus:9090

# 2단계: 결과 확인
cat reports/metrics.json | jq '.apiLatency.p95'

# 3단계: Threshold 검증
cat reports/thresholds.json

# 4단계: Gate 확인
# → threshold 위반 시 배포 차단
if [ $(cat reports/thresholds.json | jq '.failed') -gt 0 ]; then
  echo "❌ 성능 검증 실패: 배포 차단"
  exit 1
fi

# 5단계: 리포트 저장
mkdir -p artifacts
cp reports/* artifacts/kube-burner-$(date +%Y%m%d)/

# 6단계: 정리
kube-burner destroy -c cluster-density.yml
```

#### 3. 증거(Report)

| 산출물 | 설명 | 형태 |
|--------|------|------|
| **메트릭 데이터** | kube-burner 실행 결과 원본 | `reports/metrics.json` |
| **Threshold 검증** | Pass/Fail 기준 충족 여부 | `reports/thresholds.json` |
| **성능 리포트** | p50, p95, p99 통계 요약 | `reports/index.json` |

## 실무 적용 시나리오

### 시나리오 1: 클러스터 배포 후 성능 검증

**목적:** 새로 배포한 클러스터가 운영 승인 성능 기준을 만족하는지 확인

**절차:**

```bash
# 1. cluster-density 워크로드 실행
kube-burner init -c cluster-density.yml \
  --prometheus-url http://prometheus:9090

# 2. 결과 확인
p95_latency=$(cat reports/metrics.json | jq '.apiLatency.p95')

# 3. Gate 확인
if [ $(echo "$p95_latency > 100" | bc) -eq 1 ]; then
  echo "❌ p95 API latency 초과: $p95_latency ms"
  exit 1
fi

# 4. 리포트 저장
cp reports/* artifacts/kube-burner-$(date +%Y%m%d)/
```

**합격 기준:** p95 API latency < 100ms

### 시나리오 2: 노드 수용량 확인

**목적:** 노드당 최대 파드 수 확인

**절차:**

```bash
# node-density 워크로드 실행
kube-burner init -c node-density.yml \
  --prometheus-url http://prometheus:9090

# 노드당 파드 수 확인
pods_per_node=$(cat reports/metrics.json | jq '.podsPerNode')
```

**합격 기준:** 노드당 최대 100 Pod 배포 가능

## 관련 문서

- [06-검증질문-합격기준.md](./06-검증질문-합격기준.md) - 검증 질문 및 합격 기준 상세 정의
- [04-워크로드-시나리오.md](./04-워크로드-시나리오.md) - 워크로드 시나리오 선택 가이드
