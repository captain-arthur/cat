# CAT 관점에서의 Testkube 활용

Testkube는 **쿠버네티스 클러스터 인수테스트(CAT: Cluster Acceptance Test)**의 **"테스트 자동화(Test Automation)"** 카테고리에 해당하는 핵심 도구입니다.

## CAT에서의 Testkube 역할

### 카테고리 위치

| 카테고리 | 검증 질문(Assertion) | 도구 | Testkube 역할 |
|---------|---------------------|------|--------------|
| **테스트 자동화(Test Automation)** | "다양한 테스트를 자동화하여 실행할 수 있는가?" | Testkube | ✅ 핵심 도구 |

### CAT 흐름에서의 위치

```
CAT 단계별 흐름:
1. 기본 적합성(Conformance) → Sonobuoy
2. 보안 기본선(Security baseline) → Kubescape / kube-bench
3. 성능/수용량(Scale) → kube-burner / ClusterLoader2
4. 복원력(Resilience) → Chaos Mesh / LitmusChaos
5. 테스트 자동화 → Testkube ✅ ← 여기
6. 리포트/자동화 → CI 파이프라인
```

## 검증 질문(Assertion) 중심 접근

### 핵심 원칙

**"도구"가 아니라 "검증 질문(Assertion)"으로 카테고리를 고정**

- 도구는 경계가 모호하다
- 같은 도구라도 질문이 다르면 다른 카테고리에 넣어도 된다
- 필요하다면 "Primary 카테고리 1개 + Secondary 태그"로 운영

### Testkube를 사용하는 검증 질문(Assertion)

| 검증 질문(Assertion) | 테스트 방법 | 합격 기준 예시 |
|---------------------|------------|---------------|
| **"API 테스트를 자동화하여 실행할 수 있는가?"** | Postman 테스트 실행 → 결과 확인 | 테스트 실행 성공 (PASS) |
| **"성능 테스트를 자동화하여 실행할 수 있는가?"** | k6 성능 테스트 실행 → 결과 확인 | 성능 테스트 실행 성공 (PASS) |
| **"복합 테스트 워크플로우를 구성하고 실행할 수 있는가?"** | TestWorkflow 실행 → 결과 확인 | Workflow 실행 성공 (PASS) |

## CAT 기준 정의 예시

### 기준(Policy) + 절차(Procedure) + 증거(Report)

#### 1. 기준(Policy)

| 검증 질문 | 합격 기준 | 비고 |
|----------|----------|------|
| **"다양한 테스트를 Kubernetes에서 자동화할 수 있는가?"** | 테스트 실행 성공 (PASS) | 테스트 자동화 확인 |
| **"API 테스트, 성능 테스트, E2E 테스트를 자동화할 수 있는가?"** | 각 테스트 실행 성공 (PASS) | 다양한 테스트 프레임워크 지원 확인 |
| **"복합 테스트 워크플로우를 구성하고 실행할 수 있는가?"** | Workflow 실행 성공 (PASS) | TestWorkflow 기능 확인 |

#### 2. 절차(Procedure)

```bash
# CAT Testkube 실행 절차 (권장)

# 1단계: 테스트 생성
testkube create test --name my-api-test --type postman/collection --git-uri https://github.com/example/postman-collection.git

# 2단계: 테스트 실행
testkube run test my-api-test

# 3단계: 결과 확인
EXECUTION_ID=$(testkube get execution my-api-test --output json | jq -r '.results[0].id')
testkube get execution $EXECUTION_ID

# 4단계: Gate 확인
# → 테스트 실패 시 배포 차단
STATUS=$(testkube get execution $EXECUTION_ID --output json | jq -r '.result.status')
if [ "$STATUS" != "passed" ]; then
  echo "❌ 테스트 자동화 검증 실패: 배포 차단"
  exit 1
fi

# 5단계: 리포트 저장
mkdir -p artifacts
testkube download artifacts $EXECUTION_ID --output-dir ./artifacts

# 6단계: 정리
# (CLI 모드에서는 별도 정리 불필요)
```

#### 3. 증거(Report)

| 산출물 | 설명 | 형태 |
|--------|------|------|
| **TestExecution CRD** | 테스트 실행 결과 원본 | `testexecution` CRD |
| **로그** | 테스트 실행 로그 | `testkube logs` 출력 |
| **아티팩트** | 테스트 아티팩트 | 파일 다운로드 |

## 관련 문서

- [07-검증질문-합격기준.md](./07-검증질문-합격기준.md) - 검증 질문 및 합격 기준 상세 정의
- [09-CICD-통합.md](./09-CICD-통합.md) - CI/CD 파이프라인 통합 가이드
