# CAT 관점에서의 Hydrophone 활용

Hydrophone은 **쿠버네티스 클러스터 인수테스트(CAT: Cluster Acceptance Test)**의 **"기본 자격/기능(Conformance/Config)"** 카테고리에 해당하는 도구입니다.

## CAT에서의 Hydrophone 역할

### 카테고리 위치

| 카테고리 | 검증 질문(Assertion) | 도구 | Hydrophone 역할 |
|---------|---------------------|------|----------------|
| **기본 자격/기능(Conformance/Config)** | "클러스터가 공식 Kubernetes Conformance를 만족하는가?" | Hydrophone 또는 Sonobuoy | ✅ 경량 도구로 빠른 검증 |

### CAT 흐름에서의 위치

```
CAT 단계별 흐름:
1. 기본 적합성(Conformance) → Hydrophone 또는 Sonobuoy ✅ ← 여기
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

### Hydrophone을 사용하는 검증 질문(Assertion)

| 검증 질문(Assertion) | 테스트 방법 | 합격 기준 예시 |
|---------------------|------------|---------------|
| **"클러스터가 공식 Conformance를 100% 통과하는가?"** | `hydrophone run --wait` | 모든 Conformance 테스트 100% 통과 (예외 0건) |
| **"Conformance 테스트를 빠르게 실행할 수 있는가?"** | `hydrophone run --skip "\[Slow\]" --wait` | 핵심 Conformance 테스트 통과 (Slow 제외) |
| **"운영 환경에서 기본 자격을 확인했는가?"** | `hydrophone run --skip "\[Disruptive\]" --wait` | Non-disruptive Conformance 테스트 통과 (실패 0건) |
| **"표준 API 동작이 Kubernetes 표준과 차이가 없는가?"** | `hydrophone run --focus "\[Conformance\]" --wait` | Conformance 테스트 통과 |

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

**Hydrophone 역할:**
- 공식 Conformance 이미지를 사용하여 Conformance 테스트 실행
- 표준 준수 여부를 pass/fail로 확인
- CNCF Certified Kubernetes 인증 준비에 사용 가능

### Config = "우리 운영 기준에 맞게 설정돼 있는가?" (구성 점검)

**정의:** 표준 준수(Conformance)와 별개로, 이 클러스터가 **우리 조직의 운영/보안/규칙**에 맞게 세팅됐는가

**검증 항목:**
- Admission 정책(필요한 보안 정책이 켜졌나)
- ResourceQuota/LimitRange 운영 기준
- 네임스페이스 격리 방식
- Ingress 클래스/로깅/모니터링이 기본 구성됐는지

**Hydrophone 역할:**
- **Hydrophone만으로는 Config 검증 불가**
- Conformance 테스트만 지원 (플러그인 없음)
- Config 검증은 별도 도구 필요 (예: kube-bench, 커스텀 검증)

### E2E 테스트와의 관계

| 용어 | 설명 |
|------|------|
| **E2E(End-to-End) Test** | "클러스터를 실제처럼 써보는 통합 테스트 묶음" - 큰 집합 |
| **Conformance Test** | E2E 테스트 중 "표준 준수 검증"에 해당하는 부분집합 |
| **Hydrophone** | Conformance 이미지를 사용하여 E2E 테스트 실행 |

```
Kubernetes E2E Test Suite (Conformance Image)
├── Conformance Tests (표준 준수 검증) ← Hydrophone이 실행
├── Feature Tests (기능별 테스트)
├── Performance Tests (성능 테스트)
└── Disruptive Tests (파괴적 테스트)
```

## CAT 기준 정의 예시

### 기준(Policy) + 절차(Procedure) + 증거(Report)

#### 1. 기준(Policy)

| 검증 질문 | 합격 기준 | 비고 |
|----------|----------|------|
| **"클러스터가 공식 Conformance를 통과하는가?"** | Hydrophone 실행 시 **실패 0건** | 전체 Conformance 테스트 통과 |
| **"핵심 Conformance 테스트를 통과하는가?"** | Slow/Disruptive 제외 시 **실패 0건** | 빠른 검증 (운영 환경) |
| **"표준 API 동작이 정상인가?"** | Conformance 태그 테스트 **100% 통과** | 예외 없음 |

#### 2. 절차(Procedure)

```bash
# CAT Hydrophone 실행 절차 (권장)

# 1단계: 빠른 확인 (Slow 제외)
hydrophone run --skip "\[Slow\]" --wait
# → 기본 동작 확인

# 2단계: 기본 검증 (Non-Disruptive Conformance) - 운영 환경
hydrophone run --skip "\[Disruptive\]" --wait
# → 운영 환경 기본 자격 확인 (권장)

# 3단계: 결과 확인
if [ -f "e2e.log" ] && [ -f "junit_01.xml" ]; then
  # 실패 항목 확인
  fail_count=$(grep -ic "fail" e2e.log || echo "0")
  echo "실패 항목: $fail_count"
fi

# 4단계: Gate 확인
# → 실패 항목이 있으면 배포 차단 또는 알림
if [ "$fail_count" -gt 0 ]; then
  echo "❌ Conformance 검증 실패: 배포 차단"
  exit 1
fi

# 5단계: 리포트 저장
mkdir -p reports
timestamp=$(date +%Y%m%d-%H%M%S)
cp e2e.log junit_01.xml reports/hydrophone-$timestamp/

# 6단계: 정리
kubectl delete namespace conformance --ignore-not-found=true
```

#### 3. 증거(Report)

| 산출물 | 설명 | 형태 |
|--------|------|------|
| **결과 파일** | Hydrophone 실행 결과 원본 | `e2e.log`, `junit_01.xml` |
| **요약 리포트** | pass/fail, 실패 항목 정리 | `e2e.log` 분석 + 문서화 |
| **JUnit XML** | 표준 형식의 테스트 결과 | `junit_01.xml` |

**리포트 산출물 형태:**

```
hydrophone-results-YYYYMMDD/
├── e2e.log                          # 전체 테스트 실행 로그
├── junit_01.xml                     # JUnit XML 형식 결과
├── summary.md                        # 요약 리포트 (1~2페이지)
├── failed-tests.md                   # 실패 항목 상세
└── reproduction-guide.md             # 재현 방법 (명령어)
```

## 실무 적용 시나리오

### 시나리오 1: 클러스터 배포 후 인수 테스트

**목적:** 새로 배포한 클러스터가 운영 승인 기준을 만족하는지 확인

**절차:**

```bash
# 1. Hydrophone 실행 (non-disruptive 모드)
hydrophone run --skip "\[Disruptive\]" --wait

# 2. 결과 확인
fail_count=$(grep -ic "fail" e2e.log || echo "0")

# 3. Gate 확인
if [ "$fail_count" -gt 0 ]; then
  echo "❌ Conformance 검증 실패: 운영 승인 불가"
  exit 1
fi

# 4. 리포트 저장
mkdir -p reports
cp e2e.log junit_01.xml reports/hydrophone-$(date +%Y%m%d-%H%M%S)/

# 5. 정리
kubectl delete namespace conformance
```

**합격 기준:** 실패 0건

### 시나리오 2: 버전 업그레이드 후 호환성 확인

**목적:** 클러스터 버전 업그레이드 후 Conformance 유지 확인

**절차:**

```bash
# 업그레이드 전 백업 (선택적)
hydrophone run --wait
cp e2e.log junit_01.xml reports/before-upgrade-$(date +%Y%m%d)/

# ... 클러스터 업그레이드 ...

# 업그레이드 후 검증
hydrophone run --wait
fail_count=$(grep -ic "fail" e2e.log || echo "0")

# 결과 비교
if [ "$fail_count" -gt 0 ]; then
  echo "❌ 업그레이드 후 Conformance 실패"
  exit 1
fi

# 리포트 저장
cp e2e.log junit_01.xml reports/after-upgrade-$(date +%Y%m%d)/
```

**합격 기준:** 업그레이드 전과 동일한 Conformance 결과 유지

### 시나리오 3: CNCF 인증 준비

**목적:** CNCF Certified Kubernetes 인증을 위한 Conformance 검증

**절차:**

```bash
# 전체 Conformance 테스트 실행
hydrophone run --wait

# 결과 확인
if [ ! -f "e2e.log" ] || [ ! -f "junit_01.xml" ]; then
  echo "❌ 결과 파일 생성 실패"
  exit 1
fi

# CNCF 제출 요건 확인
# → e2e.log, junit_01.xml 필요
```

**합격 기준:** 전체 Conformance 테스트 통과, 결과 파일 생성

## 리스크 중심 접근

### 리스크와 Hydrophone 검증

| 리스크 | 장애 징후 | 테스트 | Hydrophone 역할 | 합격 기준 |
|--------|----------|--------|----------------|----------|
| **표준 준수 위반** | API/리소스 동작 불일치 | Conformance 테스트 | Hydrophone 실행 | Conformance 100% 통과 |
| **컴포넌트 오작동** | 핵심 기능 동작 실패 | E2E 테스트 | Hydrophone (Conformance) | Conformance 테스트 통과 |
| **업그레이드 호환성 문제** | 버전 간 동작 차이 | 버전별 Conformance 비교 | Hydrophone (업그레이드 전/후) | 동일 결과 유지 |

## Sonobuoy vs Hydrophone 선택 가이드

### CAT 관점에서의 선택

| 상황 | 권장 도구 | 이유 |
|------|----------|------|
| **빠른 Conformance 검증만 필요** | Hydrophone | 더 빠르고 단순한 실행 |
| **다양한 검증 및 로그 수집 필요** | Sonobuoy | 플러그인으로 확장 가능 |
| **경량 도구 선호** | Hydrophone | 단순하고 가벼움 |
| **커스텀 검증 항목 추가 필요** | Sonobuoy | 커스텀 플러그인 지원 |

자세한 내용은 [03-Sonobuoy-비교.md](./03-Sonobuoy-비교.md) 참조.

## 자동화 통합

### CI/CD 파이프라인 예시

```yaml
# .github/workflows/cat-hydrophone.yml (GitHub Actions 예시)
name: CAT - Hydrophone Conformance Test

on:
  workflow_dispatch
  schedule:
    - cron: '0 2 * * 0'  # 주 1회 (일요일 새벽 2시)

jobs:
  hydrophone-conformance:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.21'

      - name: Setup Hydrophone
        run: |
          go install sigs.k8s.io/hydrophone@latest
          echo "$HOME/go/bin" >> $GITHUB_PATH
          hydrophone version

      - name: Configure kubectl
        run: |
          echo "${{ secrets.KUBECONFIG }}" | base64 -d > ~/.kube/config
          kubectl version --short

      - name: Run Hydrophone
        run: |
          hydrophone run --skip "\[Disruptive\]" --wait

      - name: Check Results
        run: |
          if [ ! -f "e2e.log" ] || [ ! -f "junit_01.xml" ]; then
            echo "❌ 결과 파일 생성 실패"
            exit 1
          fi
          
          fail_count=$(grep -ic "fail" e2e.log || echo "0")
          echo "실패 항목: $fail_count"
          
          if [ "$fail_count" -gt 0 ]; then
            echo "❌ Conformance 검증 실패"
            exit 1
          fi

      - name: Upload Artifacts
        uses: actions/upload-artifact@v3
        with:
          name: hydrophone-results
          path: |
            e2e.log
            junit_01.xml

      - name: Cleanup
        if: always()
        run: kubectl delete namespace conformance --ignore-not-found=true
```

## 관련 문서

- [07-검증질문-합격기준.md](./07-검증질문-합격기준.md) - 검증 질문 및 합격 기준 상세 정의
- [09-CICD-통합.md](./09-CICD-통합.md) - CI/CD 파이프라인 통합 가이드
