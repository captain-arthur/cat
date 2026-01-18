# ClusterLoader2 POC (Proof of Concept)

로컬 Kubernetes 클러스터에서 ClusterLoader2를 실행하여 실제 동작 방식과 리포팅 차이를 확인하는 POC입니다.

## POC 목적

1. **ClusterLoader2의 리포팅 방식 확인**
   - kube-burner와의 차이점 확인
   - 내장 measurement 리포트 형식 확인

2. **JUnit XML 리포트 생성 확인**
   - kube-burner와 달리 JUnit 형식 리포트 자동 생성

## 실행 환경

| 항목 | 값 |
|------|-----|
| **클러스터 타입** | Docker Desktop Kubernetes |
| **노드** | docker-desktop |
| **Kubernetes 버전** | v1.32.2 |
| **ClusterLoader2** | 소스에서 빌드 (perf-tests 리포지토리) |

## 확인된 차이점

### 1. JUnit XML 리포트 자동 생성

**ClusterLoader2:**
- `reports/junit.xml` 자동 생성
- 테스트 결과를 표준 JUnit 형식으로 제공
- CI/CD 파이프라인에서 바로 활용 가능

**kube-burner:**
- JUnit 리포트 없음
- `reports/metrics.json`, `reports/thresholds.json` (Prometheus 있을 때만)

### 2. 내장 Measurement

**ClusterLoader2:**
- 설정 파일에 `measurements` 정의
- Prometheus 없이도 `PodStartupLatency`, `APIAvailability` 측정 가능
- 리포트에 결과 포함

**kube-burner:**
- Prometheus 없으면 메트릭 수집 불가
- `reports/` 폴더 미생성

## 실행 결과 요약

### 생성된 리포트 파일

| 파일 | 설명 |
|------|------|
| `junit.xml` | JUnit 형식 테스트 결과 리포트 ✅ |
| `generatedConfig_*.yaml` | 실행 시 사용된 실제 설정 파일 |
| `cl2-metadata.json` | 메타데이터 |

### junit.xml 예시

```xml
<testsuite name="ClusterLoaderV2" tests="0" failures="2" errors="0" time="10.05">
    <testcase name="simple-pod-test overall" classname="ClusterLoaderV2" time="10.046">
        <failure type="Failure">[설정 오류 내용]</failure>
    </testcase>
</testsuite>
```

**의미:**
- 테스트 실행 결과를 표준 형식(JUnit)으로 자동 리포트
- CI/CD 파이프라인에서 바로 파싱 가능
- kube-burner와의 **핵심 차이점**

## 실행 로그

상세 로그는 `execution-log.md` 참조.
