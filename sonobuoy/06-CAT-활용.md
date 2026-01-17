# CAT 관점에서의 Sonobuoy 활용

Sonobuoy는 **쿠버네티스 클러스터 인수테스트(CAT: Cluster Acceptance Test)**의 **"기본 자격/기능(Conformance/Config)"** 카테고리에 해당하는 핵심 도구입니다.

## CAT에서의 Sonobuoy 역할

### 카테고리 위치

| 카테고리 | 검증 질문(Assertion) | 도구 | Sonobuoy 역할 |
|---------|---------------------|------|--------------|
| **기본 자격/기능(Conformance/Config)** | "클러스터가 공식 Kubernetes Conformance를 만족하는가?" | Sonobuoy | ✅ 핵심 도구 |

### CAT 흐름에서의 위치

```
CAT 단계별 흐름:
1. 기본 적합성(Conformance) → Sonobuoy ✅ ← 여기
2. 보안 기본선(Security baseline) → kube-bench
3. 성능/수용량(Scale) → kube-burner / ClusterLoader2
4. 복원력(Resilience) → LitmusChaos
5. 리포트/자동화 → CI 파이프라인
```

## 검증 질문(Assertion) 중심 접근

### 핵심 원칙

**"도구"가 아니라 "검증 질문(Assertion)"으로 카테고리를 고정**

- 도구는 경계가 모호하다
- 같은 도구라도 질문이 다르면 다른 카테고리에 넣어도 된다
- 필요하다면 "Primary 카테고리 1개 + Secondary 태그"로 운영

### Sonobuoy를 사용하는 검증 질문(Assertion)

| 검증 질문(Assertion) | 테스트 방법 | 합격 기준 예시 |
|---------------------|------------|---------------|
| **"클러스터가 공식 Conformance를 100% 통과하는가?"** | `sonobuoy run --mode certified-conformance` | 모든 필수 Conformance 테스트 100% 통과 (예외 0건) |
| **"운영 환경에서 기본 자격을 확인했는가?"** | `sonobuoy run --mode non-disruptive-conformance` | Non-disruptive Conformance 테스트 통과 (실패 0건) |
| **"표준 API 동작/리소스 스펙/스케줄러 동작이 Kubernetes 표준과 차이가 없는가?"** | `sonobuoy run --mode non-disruptive-conformance --e2e-focus="\[Conformance\]"` | 핵심 Conformance 테스트 통과 |
| **"클러스터가 '쿠버네티스답게' 동작하는가?"** | `sonobuoy run --mode non-disruptive-conformance` | 모든 기본 리소스(Pod, Service, ConfigMap 등) 정상 동작 확인 |

## Conformance vs Config 구분

### Conformance = "쿠버네티스답게 동작하는가?" (표준 준수)

**정의:** "이 클러스터가 쿠버네티스 API/동작 규약을 제대로 만족하는가"

- Kubernetes가 정의한 **표준 동작**을 만족하는지 확인
- 결과는 **pass/fail**로 명확히 떨어짐
- "기본 자격증" 같은 느낌

**검증 항목:**
- Pod 생성/스케줄링이 정상인가
- Service/Endpoint 동작이 정상인가
- Job, ConfigMap, Secret 등 기본 리소스가 정상인가
- kubelet / API server 핵심 동작이 정상인가

**Sonobuoy 역할:**
- 공식 Conformance 테스트 실행
- 표준 준수 여부를 pass/fail로 확인
- CNCF Certified Kubernetes 인증 기준

### Config = "우리 운영 기준에 맞게 설정돼 있는가?" (구성 점검)

**정의:** 표준 준수(Conformance)와 별개로, 이 클러스터가 **우리 조직의 운영/보안/규칙**에 맞게 세팅됐는가

**검증 항목:**
- Admission 정책(필요한 보안 정책이 켜졌나)
- ResourceQuota/LimitRange 운영 기준
- 네임스페이스 격리 방식
- Ingress 클래스/로깅/모니터링이 기본 구성됐는지

**Sonobuoy 역할:**
- Sonobuoy만으로는 부족
- **커스텀 플러그인**을 통해 Config 검증 추가 가능
- 예: kube-bench 플러그인으로 보안 설정 검사

### E2E 테스트와의 관계

| 용어 | 설명 |
|------|------|
| **E2E(End-to-End) Test** | "클러스터를 실제처럼 써보는 통합 테스트 묶음" - 큰 집합 |
| **Conformance Test** | E2E 테스트 중 "표준 준수 검증"에 해당하는 부분집합 |
| **Sonobuoy** | E2E 테스트(특히 Conformance)를 실행하는 도구 |

```
Kubernetes E2E Test Suite
├── Conformance Tests (표준 준수 검증) ← Sonobuoy가 실행
├── Feature Tests (기능별 테스트)
├── Performance Tests (성능 테스트)
└── Disruptive Tests (파괴적 테스트)
```

## CAT 기준 정의 예시

### 기준(Policy) + 절차(Procedure) + 증거(Report)

#### 1. 기준(Policy)

| 검증 질문 | 합격 기준 | 비고 |
|----------|----------|------|
| **"클러스터가 공식 Conformance를 통과하는가?"** | Sonobuoy non-disruptive-conformance 모드에서 **실패 0건** | 운영 환경 기본 검증 |
| **"핵심 컴포넌트가 정상 동작하는가?"** | p0 컴포넌트(네트워킹, 스토리지, 서비스 객체) 관련 테스트 **100% 통과** | 예외 없음 |
| **"공식 인증 기준을 만족하는가?"** | certified-conformance 모드에서 **전체 테스트 통과** (테스트 환경) | CNCF 인증 준비 시 |

#### 2. 절차(Procedure)

```bash
# CAT Sonobuoy 실행 절차 (권장)

# 1단계: 빠른 확인 (Quick)
sonobuoy run --mode quick --wait
# → 기본 동작 확인

# 2단계: 기본 검증 (Non-Disruptive Conformance) - 운영 환경
sonobuoy run --mode non-disruptive-conformance --wait --timeout 7200
# → 운영 환경 기본 자격 확인 (권장)

# 3단계: 결과 확인
results=$(sonobuoy retrieve)
sonobuoy results $results

# 4단계: Gate 확인
# → 실패 항목이 있으면 배포 차단 또는 알림
if [ $(sonobuoy results $results | grep -c "Failed") -gt 0 ]; then
  echo "❌ Sonobuoy 검증 실패: 배포 차단"
  exit 1
fi

# 5단계: 리포트 저장
mv $results /path/to/artifacts/sonobuoy-$(date +%Y%m%d).tar.gz

# 6단계: 정리
sonobuoy delete --wait
```

#### 3. 증거(Report)

| 산출물 | 설명 | 형태 |
|--------|------|------|
| **결과 파일** | Sonobuoy 실행 결과 원본 | `results.tar.gz` |
| **요약 리포트** | pass/fail, 실패 항목 정리 | `sonobuoy results` 출력 |
| **상세 로그** | 플러그인별 상세 로그 | `sonobuoy logs` 출력 |
| **조치 가이드** | failed 항목별 대응 방안 | 문서 (Markdown) |

**리포트 산출물 형태:**

```
sonobuoy-results-YYYYMMDD/
├── results.tar.gz                    # 원본 결과 파일
├── summary.md                        # 요약 리포트 (1~2페이지)
├── failed-tests.md                   # 실패 항목 상세
├── logs/
│   ├── e2e.log
│   └── systemd-logs.log
└── reproduction-guide.md             # 재현 방법 (명령어/매니페스트)
```

## 실무 적용 시나리오

### 시나리오 1: 클러스터 배포 후 인수 테스트

**목적:** 새로 배포한 클러스터가 운영 승인 기준을 만족하는지 확인

**절차:**

```bash
# 1. Sonobuoy 실행 (non-disruptive-conformance 모드)
sonobuoy run --mode non-disruptive-conformance --wait --timeout 7200

# 2. 결과 확인
results=$(sonobuoy retrieve)
sonobuoy results $results

# 3. Gate 확인
if [ $(sonobuoy results $results | grep -c "Failed") -gt 0 ]; then
  echo "❌ Conformance 검증 실패: 운영 승인 불가"
  exit 1
fi

# 4. 리포트 저장
mkdir -p reports
mv $results reports/sonobuoy-$(date +%Y%m%d-%H%M%S).tar.gz

# 5. 정리
sonobuoy delete --wait
```

**합격 기준:** 실패 0건

### 시나리오 2: 버전 업그레이드 후 호환성 확인

**목적:** 클러스터 버전 업그레이드 후 Conformance 유지 확인

**절차:**

```bash
# 업그레이드 전 백업 (선택적)
sonobuoy run --mode non-disruptive-conformance --wait
results_before=$(sonobuoy retrieve)
mv $results_before reports/before-upgrade-$(date +%Y%m%d).tar.gz

# ... 클러스터 업그레이드 ...

# 업그레이드 후 검증
sonobuoy run --mode non-disruptive-conformance --wait
results_after=$(sonobuoy retrieve)

# 결과 비교
sonobuoy results $results_after

# 리포트 저장
mv $results_after reports/after-upgrade-$(date +%Y%m%d).tar.gz
```

**합격 기준:** 업그레이드 전과 동일한 Conformance 결과 유지

### 시나리오 3: CNCF 인증 준비

**목적:** CNCF Certified Kubernetes 인증을 위한 Conformance 검증

**절차:**

```bash
# 테스트 환경에서 실행 (주의: disruptive 테스트 포함)
sonobuoy run --mode certified-conformance --wait --timeout 14400

# 결과 확인
results=$(sonobuoy retrieve)
sonobuoy results $results

# 인증 요건 확인
# → 모든 Conformance 테스트 통과 필요
```

**합격 기준:** certified-conformance 모드에서 전체 테스트 통과

## 리스크 중심 접근

### 리스크와 Sonobuoy 검증

| 리스크 | 장애 징후 | 테스트 | Sonobuoy 역할 | 합격 기준 |
|--------|----------|--------|--------------|----------|
| **표준 준수 위반** | API/리소스 동작 불일치 | Conformance 테스트 | Sonobuoy 실행 | Conformance 100% 통과 |
| **컴포넌트 오작동** | 핵심 기능 동작 실패 | E2E 테스트 | Sonobuoy (특정 태그) | p0 컴포넌트 테스트 통과 |
| **업그레이드 호환성 문제** | 버전 간 동작 차이 | 버전별 Conformance 비교 | Sonobuoy (업그레이드 전/후) | 동일 결과 유지 |

## 자동화 통합

### CI/CD 파이프라인 예시

```yaml
# .github/workflows/cat-sonobuoy.yml (GitHub Actions 예시)
name: CAT - Sonobuoy Conformance Test

on:
  workflow_dispatch
  schedule:
    - cron: '0 2 * * 0'  # 주 1회 (일요일 새벽 2시)

jobs:
  sonobuoy-conformance:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Sonobuoy
        run: |
          wget https://github.com/vmware-tanzu/sonobuoy/releases/download/v0.57.3/sonobuoy_0.57.3_linux_amd64.tar.gz
          tar xzf sonobuoy_0.57.3_linux_amd64.tar.gz
          sudo mv sonobuoy /usr/local/bin/

      - name: Configure kubectl
        run: |
          # kubectl 설정 (환경에 맞게)
          echo "${{ secrets.KUBECONFIG }}" | base64 -d > ~/.kube/config

      - name: Run Sonobuoy
        run: |
          sonobuoy run --mode non-disruptive-conformance --wait --timeout 7200

      - name: Retrieve Results
        run: |
          results=$(sonobuoy retrieve)
          echo "results_file=$results" >> $GITHUB_ENV

      - name: Check Results
        run: |
          sonobuoy results $results_file
          if sonobuoy results $results_file | grep -q "Failed"; then
            echo "❌ Conformance 검증 실패"
            exit 1
          fi

      - name: Upload Artifacts
        uses: actions/upload-artifact@v3
        with:
          name: sonobuoy-results
          path: ${{ env.results_file }}

      - name: Cleanup
        run: sonobuoy delete --wait
```

## 관련 문서

- [07-검증질문-합격기준.md](./07-검증질문-합격기준.md) - 검증 질문 및 합격 기준 상세 정의
- [09-CICD-통합.md](./09-CICD-통합.md) - CI/CD 파이프라인 통합 가이드
