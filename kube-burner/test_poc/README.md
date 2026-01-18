# kube-burner POC (Proof of Concept)

로컬 Kubernetes 클러스터에서 kube-burner를 실행하여 실제 동작 방식을 확인하는 POC입니다.

## POC 목적

1. **어떤 성능을 어떤 방식으로 테스트하는가?**
   - 어떤 메트릭을 수집하는지
   - 어떤 방식으로 부하를 생성하는지

2. **결과를 어떻게 리포팅하는가?**
   - 결과 파일 형식
   - 리포트 내용

3. **무엇을 기준으로 통과/실패를 판단하는가?**
   - Threshold 설정 방법
   - Pass/Fail 기준

## 문서 구조

### 실행 전 확인
- [설치-이슈.md](./설치-이슈.md) - 설치 방법 및 이슈 해결

### 실행 결과
- [02-실행-결과.md](./02-실행-결과.md) - 실제 테스트 실행 결과
- [03-결과-분석.md](./03-결과-분석.md) - 결과 분석 및 해석

### 기타
- [execution-log.md](./execution-log.md) - 실행 과정 상세 로그
- [kube-burner-*.log](./) - kube-burner 실행 로그 파일

## 클러스터 정보

| 항목 | 값 |
|------|-----|
| **클러스터 타입** | Docker Desktop Kubernetes |
| **노드** | docker-desktop |
| **Kubernetes 버전** | v1.32.2 |
| **kube-burner 버전** | v2.2.0 |

## POC 진행 상태

### ✅ 완료
- [x] kube-burner 설치 (v2.2.0)
- [x] 실행 전 확인
- [x] 간단한 워크로드 실행
- [x] 결과 확인 및 문서화

### 실행 결과 요약

| 항목 | 결과 |
|------|------|
| **실행 상태** | ✅ 성공 |
| **실행 시간** | 약 3초 (Pre-load 1분 제외) |
| **생성된 네임스페이스** | 10개 (kube-burner-test-0 ~ kube-burner-test-9) |
| **생성된 Pod** | 10개 (각 네임스페이스당 1개) |
| **실행 UUID** | 38aa0c0b-dfad-4592-b10e-afd51d5f1f43 |

**상세 결과:** [02-실행-결과.md](./02-실행-결과.md) 참조  
**결과 분석:** [03-결과-분석.md](./03-결과-분석.md) 참조

## ⚠️ 중요: 이 POC에 대해

이 POC는 **제가 직접 만든 커스텀 테스트**입니다:
- **실행한 테스트**: `simple-test-v2.yml` (직접 작성한 간단한 Pod 생성 테스트)
- **공식 시나리오 아님**: kube-burner의 공식 시나리오(`cluster-density`, `node-density` 등)는 별도로 제공됩니다
- **POC 목적**: kube-burner의 기본 동작 방식과 결과 리포팅 방식을 이해하기 위함

**실제 실행:**
- 10개 Pod 생성 (각 네임스페이스당 1개)
- 공식 시나리오와는 다름 (예: cluster-density는 수백~수천 개 리소스 생성)

## 핵심 질문에 대한 답변

### 1. 어떤 성능을 테스트했는가?

**현재 실행:**
- 기본 리소스 생성 테스트 (Pod 생성 동작 확인)
- 반복 실행 테스트 (10번 반복)
- 자동 정리 테스트 (리소스 정리 동작 확인)

**미수집 메트릭 (Prometheus 필요):**
- Pod latency (Pod 생성 → Ready 시간)
- API latency (API Server 응답 시간)
- Resource usage (CPU/메모리 사용량)

### 2. 어떤 방식으로 테스트했는가?

**테스트 방식:**
1. **Pre-load**: 이미지 Pull을 위한 DaemonSet 생성 (약 1분)
2. **Job 실행**: `jobIterations: 10`만큼 반복, 각 iteration마다 네임스페이스 + Pod 생성
3. **완료 대기**: Pod Ready 상태까지 자동 대기
4. **자동 정리**: 실행 완료 후 리소스 정리

### 3. 결과를 어떻게 리포팅하는가?

**콘솔 출력:**
- 실행 로그를 콘솔에 출력
- iteration 진행 상황, 완료 시간 등

**로그 파일:**
- `kube-burner-{UUID}.log`: 실행 로그 저장

**리포트 파일 (Prometheus 있을 때):**
- `reports/metrics.json`: 수집된 메트릭 데이터
- `reports/thresholds.json`: Threshold 검증 결과
- `reports/index.json`: 인덱스 정보

**현재 상태:** Prometheus 없어서 리포트 파일(`reports/`) 미생성

### 4. 무엇을 기준으로 통과/실패를 판단하는가?

**현재 실행 (Prometheus 없음):**
- 리소스 생성 성공 여부
- 실행 완료 여부 (에러 없이 완료)

**Threshold 검증 (Prometheus 있을 때):**
- YAML 설정 파일에서 임계치 정의
- 예: `p95 API latency >= 100ms` → Fail
- 결과는 `reports/thresholds.json`에 저장

자세한 내용은 [03-결과-분석.md](./03-결과-분석.md) 참조.
