# Hydrophone vs Sonobuoy 비교

Hydrophone과 Sonobuoy는 모두 Kubernetes Conformance 테스트를 실행하는 도구이지만, 목적과 접근 방식이 다릅니다.

## 개요 비교

| 항목 | Hydrophone | Sonobuoy |
|------|-----------|----------|
| **목적** | Conformance 테스트 실행에 집중 (경량) | 다양한 플러그인 및 데이터 수집 지원 (풀 기능) |
| **포지션** | 간단하고 빠른 Conformance 검증 | 종합적인 클러스터 진단 및 검증 |
| **등장 시기** | 2024년 (새로운 도구) | 2017년부터 (기존 도구) |
| **설치** | `go install` (단일 바이너리) | 바이너리 다운로드 또는 패키지 관리자 |

## 기능 비교

### 설치 및 실행

| 항목 | Hydrophone | Sonobuoy |
|------|-----------|----------|
| **설치 방법** | `go install sigs.k8s.io/hydrophone@latest` | 바이너리 다운로드, Homebrew 등 |
| **실행 명령어** | `hydrophone run` | `sonobuoy run` |
| **실행 방식** | 단일 Pod 실행 | Aggregator + 플러그인 실행 |
| **네임스페이스** | `conformance` | `sonobuoy` |

### 기능 범위

| 기능 | Hydrophone | Sonobuoy |
|------|-----------|----------|
| **Conformance 테스트** | ✅ 지원 (공식 이미지 사용) | ✅ 지원 (e2e 플러그인) |
| **플러그인 구조** | ❌ 없음 | ✅ 다양한 플러그인 지원 |
| **로그 수집** | ❌ 없음 | ✅ Systemd Logs 플러그인 |
| **커스텀 검증** | ❌ 없음 | ✅ 커스텀 플러그인 작성 가능 |
| **데이터 수집** | ❌ 없음 | ✅ 다양한 진단 데이터 수집 |

### 결과 출력

| 항목 | Hydrophone | Sonobuoy |
|------|-----------|----------|
| **결과 형식** | `e2e.log`, `junit_01.xml` | `results.tar.gz` (다양한 플러그인 결과) |
| **출력 위치** | 현재 디렉토리 | 다운로드한 tar.gz 파일 |
| **리포트 형식** | 단순 (로그 + JUnit XML) | 종합 (다양한 플러그인 결과) |

## 실행 속도 비교

| 항목 | Hydrophone | Sonobuoy |
|------|-----------|----------|
| **시작 시간** | 빠름 (단순 구조) | 상대적으로 느림 (Aggregator + 플러그인 설정) |
| **실행 시간** | Conformance 테스트만 실행 | 플러그인에 따라 더 길 수 있음 |
| **리소스 사용** | 낮음 (단일 Pod) | 상대적으로 높음 (여러 Pod) |

## 사용 사례별 선택 가이드

### Hydrophone을 선택하는 경우

| 상황 | 이유 |
|------|------|
| **빠른 Conformance 검증만 필요** | 단순하고 빠른 실행 |
| **플러그인 기능 불필요** | Conformance 테스트만으로 충분 |
| **경량 도구 선호** | 최소한의 리소스 사용 |
| **Go 환경이 있는 경우** | `go install`로 쉽게 설치 가능 |

**예시:**
```bash
# 빠른 Conformance 검증
hydrophone run --wait
```

### Sonobuoy를 선택하는 경우

| 상황 | 이유 |
|------|------|
| **다양한 검증이 필요** | 플러그인으로 확장 가능 |
| **로그 수집 필요** | Systemd Logs 등 다양한 로그 수집 |
| **커스텀 검증 필요** | 커스텀 플러그인 작성 가능 |
| **종합적인 리포트 필요** | 다양한 플러그인 결과를 통합 리포트로 |
| **패키지 관리자로 설치 선호** | Homebrew 등 패키지 관리자 사용 가능 |

**예시:**
```bash
# 종합적인 클러스터 진단
sonobuoy run --mode non-disruptive-conformance --wait
```

## 기술적 차이

### 아키텍처

| 항목 | Hydrophone | Sonobuoy |
|------|-----------|----------|
| **구성 요소** | 단일 바이너리 + Pod | CLI + Aggregator + 플러그인 |
| **이미지** | 공식 Conformance 이미지 | 커스텀 플러그인 이미지 사용 가능 |
| **실행 구조** | 단순 (직접 Pod 실행) | 복잡 (Aggregator가 플러그인 관리) |

### 결과 수집

| 항목 | Hydrophone | Sonobuoy |
|------|-----------|----------|
| **수집 방식** | Pod 로그 직접 수집 | Aggregator가 플러그인 결과 취합 |
| **결과 형식** | 표준 (e2e.log, junit.xml) | 통합 (results.tar.gz) |
| **리포트 생성** | 사용자가 직접 처리 | Sonobuoy가 자동 생성 |

## CAT 관점에서의 선택

### 검증 질문(Assertion)에 따른 선택

| 검증 질문 | 권장 도구 | 이유 |
|----------|----------|------|
| **"클러스터가 공식 Conformance를 만족하는가?"** | Hydrophone 또는 Sonobuoy | 둘 다 Conformance 테스트 지원 |
| **"Conformance 테스트를 빠르게 실행할 수 있는가?"** | Hydrophone | 더 빠르고 단순한 실행 |
| **"다양한 진단 데이터를 수집할 수 있는가?"** | Sonobuoy | 플러그인으로 확장 가능 |
| **"커스텀 검증 항목을 추가할 수 있는가?"** | Sonobuoy | 커스텀 플러그인 지원 |

### CAT 흐름에서의 위치

| 단계 | Hydrophone | Sonobuoy |
|------|-----------|----------|
| **CAT 1단계: Conformance** | ✅ 적합 (빠른 검증) | ✅ 적합 (종합 검증) |
| **CAT 2단계: Security** | ❌ 부적합 (플러그인 없음) | ✅ 적합 (kube-bench 플러그인 등) |
| **CAT 3단계: Performance** | ❌ 부적합 | ❌ 부적합 (별도 도구 필요) |
| **CAT 4단계: Resilience** | ❌ 부적합 | ❌ 부적합 (별도 도구 필요) |

## 마이그레이션 고려 사항

### Hydrophone에서 Sonobuoy로

| 항목 | 고려 사항 |
|------|----------|
| **플러그인 필요** | Sonobuoy로 마이그레이션하여 플러그인 추가 |
| **리포트 형식** | `results.tar.gz` 형식으로 변경 필요 |
| **실행 명령어** | `hydrophone run` → `sonobuoy run` |
| **네임스페이스** | `conformance` → `sonobuoy` |

### Sonobuoy에서 Hydrophone으로

| 항목 | 고려 사항 |
|------|----------|
| **플러그인 기능** | 플러그인 기능은 사용 불가 |
| **리포트 형식** | `results.tar.gz` → `e2e.log`, `junit_01.xml` |
| **실행 명령어** | `sonobuoy run` → `hydrophone run` |
| **네임스페이스** | `sonobuoy` → `conformance` |

## 권장 사항

### 기본 가이드라인

1. **빠른 Conformance 검증만 필요한 경우**: Hydrophone 권장
2. **다양한 검증과 리포트가 필요한 경우**: Sonobuoy 권장
3. **CNCF 인증 준비**: 둘 다 사용 가능 (선택에 따라)

### CAT 통합 시

- **경량 CAT**: Hydrophone 사용 (Conformance 검증만)
- **종합 CAT**: Sonobuoy 사용 (다양한 검증 포함)

## 관련 문서

- [01-개요.md](./01-개요.md) - Hydrophone 개요
- [04-기본사용법.md](./04-기본사용법.md) - Hydrophone 사용법
- [../sonobuoy/01-개요.md](../sonobuoy/01-개요.md) - Sonobuoy 개요
