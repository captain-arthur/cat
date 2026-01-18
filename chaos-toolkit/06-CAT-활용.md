# CAT 관점에서의 Chaos Toolkit 활용

Chaos Toolkit은 **쿠버네티스 클러스터 인수테스트(CAT: Cluster Acceptance Test)**의 **"복원력/장애 대응(Resilience/Fault Tolerance)"** 카테고리에 해당하는 핵심 도구입니다.

## CAT에서의 Chaos Toolkit 역할

### 카테고리 위치

| 카테고리 | 검증 질문(Assertion) | 도구 | Chaos Toolkit 역할 |
|---------|---------------------|------|-------------------|
| **복원력/장애 대응(Resilience/Fault Tolerance)** | "클러스터가 장애 상황에서도 정상 동작을 유지하는가?" | Chaos Toolkit | ✅ 핵심 도구 |

### CAT 흐름에서의 위치

```
CAT 단계별 흐름:
1. 기본 적합성(Conformance) → Sonobuoy
2. 보안 기본선(Security baseline) → kube-bench
3. 성능/수용량(Scale) → kube-burner / ClusterLoader2
4. 복원력(Resilience) → Chaos Toolkit ✅ ← 여기
5. 리포트/자동화 → CI 파이프라인
```

## 검증 질문(Assertion) 중심 접근

### 핵심 원칙

**"도구"가 아니라 "검증 질문(Assertion)"으로 카테고리를 고정**

- 도구는 경계가 모호하다
- 같은 도구라도 질문이 다르면 다른 카테고리에 넣어도 된다
- 필요하다면 "Primary 카테고리 1개 + Secondary 태그"로 운영

### Chaos Toolkit를 사용하는 검증 질문(Assertion)

| 검증 질문(Assertion) | 테스트 방법 | 합격 기준 예시 |
|---------------------|------------|---------------|
| **"Pod가 죽었을 때 자동으로 복구되는가?"** | Pod 삭제 → Steady-State Hypothesis 검증 | 실험 후 Pod 상태가 "Running"으로 복구 |
| **"노드 장애 시 워크로드가 다른 노드로 이동하는가?"** | 노드 종료 → Pod 재스케줄링 확인 | Pod가 다른 노드에 재스케줄링됨 |
| **"네트워크 분할 상황에서도 서비스가 정상 동작하는가?"** | 네트워크 분할 시뮬레이션 → 서비스 응답 확인 | 서비스가 정상 응답 유지 |

## 리스크 중심 접근

### 리스크와 Chaos Toolkit 검증

| 리스크 | 장애 징후 | 테스트 | Chaos Toolkit 역할 | 합격 기준 |
|--------|----------|--------|-------------------|----------|
| **Pod 장애** | Pod가 죽었을 때 서비스 중단 | Pod 삭제 → 자동 복구 확인 | Steady-State Hypothesis 검증 | Pod 상태가 "Running"으로 복구 |
| **노드 장애** | 노드 장애 시 워크로드 중단 | 노드 종료 → Pod 재스케줄링 확인 | 노드 장애 시뮬레이션 + Probe | Pod가 다른 노드에 재스케줄링됨 |
| **네트워크 지연** | 네트워크 지연 시 서비스 응답 저하 | 네트워크 지연 주입 → 응답 시간 확인 | 네트워크 지연 시뮬레이션 + Probe | 응답 시간이 허용 범위 내 |

## CAT 기준 정의 예시

### 기준(Policy) + 절차(Procedure) + 증거(Report)

#### 1. 기준(Policy)

| 검증 질문 | 합격 기준 | 비고 |
|----------|----------|------|
| **"Pod 삭제 후 자동 복구가 요구 시간 내에 완료되는가?"** | Pod 삭제 후 60초 이내에 "Running" 상태로 복구 | 고가용성 확인 |
| **"노드 장애 시 Pod가 다른 노드로 재스케줄링되는가?"** | 노드 장애 후 5분 이내에 Pod가 다른 노드에 스케줄링됨 | 노드 장애 대응 확인 |
| **"서비스가 네트워크 지연 상황에서도 정상 응답하는가?"** | 네트워크 지연 주입 후 HTTP 200 응답 유지 | 네트워크 복원력 확인 |

#### 2. 절차(Procedure)

```bash
# CAT Chaos Toolkit 실행 절차 (권장)

# 1단계: 실험 파일 검증
chaos validate experiment.json

# 2단계: 실험 실행
chaos run experiment.json --journal-path journal.json

# 3단계: Journal 확인
cat journal.json | jq '.status'

# 4단계: Steady-State Hypothesis 검증
cat journal.json | jq '.steady-state-hypothesis.deviated'

# 5단계: Gate 확인
# → Steady-State Hypothesis 위반 시 배포 차단
if [ $(cat journal.json | jq '.steady-state-hypothesis.deviated') = "true" ]; then
  echo "❌ 복원력 검증 실패: 배포 차단"
  exit 1
fi

# 6단계: 리포트 저장
mkdir -p artifacts
chaos report journal.json artifacts/report-$(date +%Y%m%d).html

# 7단계: 정리
# (필요 시 Rollback 실행)
```

#### 3. 증거(Report)

| 산출물 | 설명 | 형태 |
|--------|------|------|
| **Journal 파일** | Chaos Toolkit 실행 결과 원본 | `chaos-report.json` 또는 `journal.json` |
| **HTML 리포트** | 실험 결과 시각화 | `report.html` |
| **Steady-State Hypothesis 결과** | 실험 전/후 상태 검증 결과 | Journal 파일 내 `steady-state-hypothesis` |

## 실무 적용 시나리오

### 시나리오 1: Pod 자동 복구 테스트

**목적:** Pod 삭제 시 자동으로 복구되는지 확인

**절차:**

```bash
# 1. 실험 실행
chaos run pod-resilience.json --journal-path journal.json

# 2. 결과 확인
cat journal.json | jq '.status'

# 3. Steady-State Hypothesis 확인
cat journal.json | jq '.steady-state-hypothesis'

# 4. Gate 확인
if [ $(cat journal.json | jq '.status') != "completed" ]; then
  echo "❌ Pod 자동 복구 검증 실패"
  exit 1
fi
```

**합격 기준:** 실험 전/후 Pod 상태가 "Running"으로 유지

### 시나리오 2: 노드 장애 대응 테스트

**목적:** 노드 장애 시 워크로드가 다른 노드로 이동하는지 확인

**절차:**

```bash
# 1. 실험 실행
chaos run node-failure.json --journal-path journal.json

# 2. 결과 확인
cat journal.json | jq '.status'

# 3. Steady-State Hypothesis 확인
cat journal.json | jq '.steady-state-hypothesis.deviated'

# 4. Gate 확인
if [ $(cat journal.json | jq '.steady-state-hypothesis.deviated') = "true" ]; then
  echo "❌ 노드 장애 대응 검증 실패"
  exit 1
fi
```

**합격 기준:** 노드 장애 후 Pod가 다른 노드에 재스케줄링됨

### 시나리오 3: 네트워크 지연 복원력 테스트

**목적:** 네트워크 지연 상황에서도 서비스가 정상 응답하는지 확인

**절차:**

```bash
# 1. 실험 실행
chaos run network-latency.json --journal-path journal.json

# 2. 결과 확인
cat journal.json | jq '.status'

# 3. Steady-State Hypothesis 확인
cat journal.json | jq '.steady-state-hypothesis.probes[*].status'

# 4. Gate 확인
if [ $(cat journal.json | jq '.steady-state-hypothesis.deviated') = "true" ]; then
  echo "❌ 네트워크 복원력 검증 실패"
  exit 1
fi
```

**합격 기준:** 네트워크 지연 주입 후 서비스가 정상 응답 유지

## 자동화 통합

### CI/CD 파이프라인 예시

```yaml
# .github/workflows/cat-chaos-toolkit.yml (GitHub Actions 예시)
name: CAT - Chaos Toolkit Resilience Test

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

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.8'

      - name: Install Chaos Toolkit
        run: |
          pip install -U chaostoolkit chaostoolkit-kubernetes

      - name: Configure kubectl
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG }}
        run: |
          echo "$KUBECONFIG" | base64 -d > ~/.kube/config

      - name: Run Experiment
        run: |
          chaos run experiment.json --journal-path journal.json

      - name: Check Results
        run: |
          STATUS=$(cat journal.json | jq -r '.status')
          if [ "$STATUS" != "completed" ]; then
            echo "❌ 복원력 검증 실패"
            exit 1
          fi
          echo "✅ 복원력 검증 성공"

      - name: Generate Report
        run: |
          chaos report journal.json report.html

      - name: Upload Artifacts
        uses: actions/upload-artifact@v3
        with:
          name: chaos-toolkit-results
          path: |
            journal.json
            report.html
```

## 관련 문서

- [07-검증질문-합격기준.md](./07-검증질문-합격기준.md) - 검증 질문 및 합격 기준 상세 정의
- [09-CICD-통합.md](./09-CICD-통합.md) - CI/CD 파이프라인 통합 가이드
