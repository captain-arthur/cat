# CAT 관점에서의 kube-bench 활용

kube-bench는 **쿠버네티스 클러스터 인수테스트(CAT: Cluster Acceptance Test)**의 **"보안 기본선(Security baseline)"** 카테고리에 해당하는 핵심 도구입니다.

## CAT에서의 kube-bench 역할

### 카테고리 위치

| 카테고리 | 검증 질문(Assertion) | 도구 | kube-bench 역할 |
|---------|---------------------|------|--------------|
| **보안 기본선(Security baseline)** | "클러스터 보안 설정이 CIS 기준을 만족하는가?" | kube-bench | ✅ 핵심 도구 |

### CAT 흐름에서의 위치

```
CAT 단계별 흐름:
1. 기본 적합성(Conformance) → Sonobuoy
2. 보안 기본선(Security baseline) → kube-bench ✅ ← 여기
3. 성능/수용량(Scale) → kube-burner / ClusterLoader2
4. 복원력(Resilience) → Chaos Mesh / LitmusChaos
5. 리포트/자동화 → CI 파이프라인
```

## 검증 질문(Assertion) 중심 접근

### 핵심 원칙

**"도구"가 아니라 "검증 질문(Assertion)"으로 카테고리를 고정**

- 도구는 경계가 모호하다
- 같은 도구라도 질문이 다르면 다른 카테고리에 넣어도 된다
- 필요하다면 "Primary 카테고리 1개 + Secondary 태그"로 운영

### kube-bench를 사용하는 검증 질문(Assertion)

| 검증 질문(Assertion) | 테스트 방법 | 합격 기준 예시 |
|---------------------|------------|---------------|
| **"클러스터 보안 설정이 CIS 기준을 만족하는가?"** | CIS Benchmark 검사 → FAIL 항목 확인 | FAIL 항목 0건 (또는 허용 범위 내) |
| **"마스터 노드 보안 설정이 적절한가?"** | 마스터 노드 검사 → FAIL 항목 확인 | FAIL 항목 0건 |
| **"워커 노드 보안 설정이 적절한가?"** | 워커 노드 검사 → FAIL 항목 확인 | FAIL 항목 0건 |

## CAT 기준 정의 예시

### 기준(Policy) + 절차(Procedure) + 증거(Report)

#### 1. 기준(Policy)

| 검증 질문 | 합격 기준 | 비고 |
|----------|----------|------|
| **"CIS Benchmark 준수 여부가 요구 수준인가?"** | FAIL 항목 0건 (또는 허용 범위 내) | 보안 설정 확인 |
| **"마스터 노드 보안 설정이 적절한가?"** | 마스터 노드 FAIL 항목 0건 | Control Plane 보안 확인 |
| **"워커 노드 보안 설정이 적절한가?"** | 워커 노드 FAIL 항목 0건 | Worker Node 보안 확인 |

#### 2. 절차(Procedure)

```bash
# CAT kube-bench 실행 절차 (권장)

# 1단계: 마스터 노드 검사
kube-bench run --targets master --json --output master-results.json

# 2단계: 워커 노드 검사
kube-bench run --targets node --json --output node-results.json

# 3단계: FAIL 항목 확인
FAIL_COUNT=$(cat master-results.json | jq '[.Controls[] | select(.result == "FAIL")] | length')
echo "FAIL 항목 수: $FAIL_COUNT"

# 4단계: Gate 확인
# → FAIL 항목이 있으면 배포 차단
if [ "$FAIL_COUNT" -gt 0 ]; then
  echo "❌ 보안 검증 실패: 배포 차단"
  exit 1
fi

# 5단계: 리포트 저장
mkdir -p artifacts
cp master-results.json node-results.json artifacts/

# 6단계: 정리
# (CLI 모드에서는 별도 정리 불필요)
```

#### 3. 증거(Report)

| 산출물 | 설명 | 형태 |
|--------|------|------|
| **JSON 리포트** | 검사 결과 원본 | `.json` 파일 |
| **JUnit XML** | CI/CD 통합용 | `.xml` 파일 |
| **stdout 출력** | 터미널 출력 | 텍스트 출력 |

## 관련 문서

- [07-검증질문-합격기준.md](./07-검증질문-합격기준.md) - 검증 질문 및 합격 기준 상세 정의
- [09-CICD-통합.md](./09-CICD-통합.md) - CI/CD 파이프라인 통합 가이드
