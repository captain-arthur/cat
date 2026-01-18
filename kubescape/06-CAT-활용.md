# CAT 관점에서의 Kubescape 활용

Kubescape는 **쿠버네티스 클러스터 인수테스트(CAT: Cluster Acceptance Test)**의 **"보안 기본선(Security baseline)"** 카테고리에 해당하는 핵심 도구입니다.

## CAT에서의 Kubescape 역할

### 카테고리 위치

| 카테고리 | 검증 질문(Assertion) | 도구 | Kubescape 역할 |
|---------|---------------------|------|--------------|
| **보안 기본선(Security baseline)** | "클러스터 보안 설정이 기준을 만족하는가?" | Kubescape | ✅ 핵심 도구 |

### CAT 흐름에서의 위치

```
CAT 단계별 흐름:
1. 기본 적합성(Conformance) → Sonobuoy
2. 보안 기본선(Security baseline) → Kubescape ✅ ← 여기
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

### Kubescape를 사용하는 검증 질문(Assertion)

| 검증 질문(Assertion) | 테스트 방법 | 합격 기준 예시 |
|---------------------|------------|---------------|
| **"클러스터 보안 설정이 CIS 기준을 만족하는가?"** | CIS Benchmark 스캔 → 통과 비율 확인 | CIS Benchmark 80% 이상 통과 |
| **"컨테이너 이미지에 Critical 취약점이 없는가?"** | 이미지 스캔 → 취약점 심각도 확인 | Critical 취약점 0건 |
| **"모든 워크로드에 보안 설정이 적용되어 있는가?"** | 프레임워크 스캔 → 컨트롤 통과 확인 | 필수 컨트롤 100% 통과 |

## CAT 기준 정의 예시

### 기준(Policy) + 절차(Procedure) + 증거(Report)

#### 1. 기준(Policy)

| 검증 질문 | 합격 기준 | 비고 |
|----------|----------|------|
| **"CIS Benchmark 준수 비율이 요구 수준인가?"** | CIS Benchmark 80% 이상 통과 | 보안 설정 확인 |
| **"Critical 취약점이 없는가?"** | Critical 취약점 0건 | 이미지 보안 확인 |
| **"필수 보안 컨트롤이 통과하는가?"** | 필수 컨트롤 100% 통과 | 보안 정책 준수 확인 |

#### 2. 절차(Procedure)

```bash
# CAT Kubescape 실행 절차 (권장)

# 1단계: 프레임워크 스캔
kubescape scan framework cis --format json --output cis-scan.json

# 2단계: 결과 확인
SCORE=$(cat cis-scan.json | jq -r '.summary.score')
echo "CIS Benchmark 점수: $SCORE"

# 3단계: Gate 확인
# → 합격 기준 미달 시 배포 차단
if (( $(echo "$SCORE < 80" | bc -l) )); then
  echo "❌ 보안 검증 실패: 배포 차단"
  exit 1
fi

# 4단계: 리포트 저장
mkdir -p artifacts
cp cis-scan.json artifacts/kubescape-$(date +%Y%m%d).json

# 5단계: 정리
# (CLI 모드에서는 별도 정리 불필요)
```

#### 3. 증거(Report)

| 산출물 | 설명 | 형태 |
|--------|------|------|
| **JSON 리포트** | 스캔 결과 원본 | `.json` 파일 |
| **JUnit XML** | CI/CD 통합용 | `.xml` 파일 |
| **HTML 리포트** | 시각화 리포트 | `.html` 파일 |
| **Summary** | 점수 및 통과 비율 | JSON 내 `summary` 필드 |

## 관련 문서

- [07-검증질문-합격기준.md](./07-검증질문-합격기준.md) - 검증 질문 및 합격 기준 상세 정의
- [09-CICD-통합.md](./09-CICD-통합.md) - CI/CD 파이프라인 통합 가이드
