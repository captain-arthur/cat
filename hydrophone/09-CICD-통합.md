# CI/CD 통합 가이드

Hydrophone을 CI/CD 파이프라인에 통합하여 자동화된 클러스터 인수 테스트를 수행하는 방법을 설명합니다.

## CI/CD 통합 개요

### 목적

- **자동화**: 클러스터 배포 시 자동으로 Conformance 검증 수행
- **Gate 역할**: 검증 실패 시 배포 차단
- **리포트 관리**: 결과를 자동으로 저장 및 추적
- **지속적 검증**: 정기적으로 클러스터 상태 확인

### 통합 위치

```
CI/CD 파이프라인 흐름:
1. 코드 빌드
2. 이미지 빌드
3. 클러스터 배포
4. Hydrophone Conformance 검증 ← 여기 (CAT 1단계)
5. kube-bench 보안 검증 (CAT 2단계)
6. 성능/수용량 테스트 (CAT 3단계)
7. 복원력 테스트 (CAT 4단계)
8. 운영 승인
```

## GitHub Actions 예시

### 기본 워크플로우

```yaml
# .github/workflows/cat-hydrophone.yml
name: CAT - Hydrophone Conformance Test

on:
  workflow_dispatch:
    inputs:
      skip_disruptive:
        description: 'Disruptive 테스트 제외'
        required: false
        default: 'true'
        type: boolean
  schedule:
    - cron: '0 2 * * 0'  # 주 1회 (일요일 새벽 2시)
  push:
    branches:
      - main
    paths:
      - 'cluster/**'

jobs:
  hydrophone-conformance:
    runs-on: ubuntu-latest
    timeout-minutes: 180  # 3시간 타임아웃
    
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
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG }}
        run: |
          echo "$KUBECONFIG" | base64 -d > ~/.kube/config
          kubectl version --short
          kubectl get nodes

      - name: Run Hydrophone
        id: run
        run: |
          SKIP_OPT=""
          if [ "${{ github.event.inputs.skip_disruptive }}" == "true" ]; then
            SKIP_OPT="--skip '\[Disruptive\]'"
          fi
          
          echo "Hydrophone 실행 중..."
          hydrophone run $SKIP_OPT --wait
          
          # 결과 파일 확인
          if [ -f "e2e.log" ] && [ -f "junit_01.xml" ]; then
            echo "results_exist=true" >> $GITHUB_OUTPUT
          else
            echo "results_exist=false" >> $GITHUB_OUTPUT
          fi

      - name: Check Results
        id: check
        run: |
          if [ "${{ steps.run.outputs.results_exist }}" != "true" ]; then
            echo "❌ 결과 파일 생성 실패"
            echo "conformance_passed=false" >> $GITHUB_OUTPUT
            exit 1
          fi
          
          # 실패 항목 확인
          fail_count=$(grep -ic "fail" e2e.log || echo "0")
          echo "실패 항목: $fail_count"
          
          if [ "$fail_count" -gt 0 ]; then
            echo "conformance_passed=false" >> $GITHUB_OUTPUT
            echo "❌ Conformance 검증 실패"
            exit 1
          else
            echo "conformance_passed=true" >> $GITHUB_OUTPUT
            echo "✅ Conformance 검증 성공"
          fi

      - name: Generate Summary Report
        if: always()
        run: |
          mkdir -p reports
          
          # 결과 파일 복사
          cp e2e.log junit_01.xml reports/ 2>/dev/null || true
          
          # 요약 리포트 생성
          cat > reports/summary.md << EOF
          # Hydrophone Conformance Test 결과 요약
          
          **실행 일시:** $(date '+%Y-%m-%d %H:%M:%S %Z')
          **워크플로우:** ${{ github.workflow }}
          **커밋:** ${{ github.sha }}
          **Disruptive 제외:** ${{ github.event.inputs.skip_disruptive }}
          
          ## 전체 결과
          
          EOF
          
          if [ -f "e2e.log" ]; then
            total=$(grep -c "Test:" e2e.log || echo "0")
            passed=$(grep -ic "PASS" e2e.log || echo "0")
            failed=$(grep -ic "FAIL" e2e.log || echo "0")
            
            cat >> reports/summary.md << EOF
          - **전체 테스트:** $total
          - **성공:** $passed
          - **실패:** $failed
          
          EOF
          fi
          
          # 합격 여부
          if [ "${{ steps.check.outputs.conformance_passed }}" == "true" ]; then
            echo "✅ **합격** - 모든 Conformance 테스트 통과" >> reports/summary.md
          else
            echo "❌ **불합격** - 실패 항목 존재" >> reports/summary.md
          fi
          
          cat reports/summary.md

      - name: Upload Artifacts
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: hydrophone-results-${{ github.run_number }}
          path: |
            reports/
            e2e.log
            junit_01.xml
          retention-days: 90

      - name: Cleanup
        if: always()
        run: |
          kubectl delete namespace conformance --ignore-not-found=true || true

      - name: Gate Check
        if: steps.check.outputs.conformance_passed != 'true'
        run: |
          echo "❌ Hydrophone Conformance 검증 실패: 배포 차단"
          exit 1
```

### 간단한 버전

```yaml
# .github/workflows/hydrophone-simple.yml
name: Hydrophone Quick Check

on:
  workflow_dispatch

jobs:
  hydrophone:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.21'
      
      - name: Setup Hydrophone
        run: |
          go install sigs.k8s.io/hydrophone@latest
          echo "$HOME/go/bin" >> $GITHUB_PATH
      
      - name: Configure kubectl
        run: echo "${{ secrets.KUBECONFIG }}" | base64 -d > ~/.kube/config
      
      - name: Run Hydrophone
        run: hydrophone run --skip "\[Disruptive\]" --wait
      
      - name: Check Results
        run: |
          if [ ! -f "e2e.log" ] || [ ! -f "junit_01.xml" ]; then
            echo "❌ 결과 파일 생성 실패"
            exit 1
          fi
          
          fail_count=$(grep -ic "fail" e2e.log || echo "0")
          if [ "$fail_count" -gt 0 ]; then
            echo "❌ Conformance 검증 실패"
            exit 1
          fi
      
      - name: Cleanup
        if: always()
        run: kubectl delete namespace conformance --ignore-not-found=true
```

## GitLab CI 예시

```yaml
# .gitlab-ci.yml
stages:
  - deploy
  - cat

cat-hydrophone:
  stage: cat
  image: golang:1.21
  before_script:
    - go install sigs.k8s.io/hydrophone@latest
    - export PATH=$PATH:$(go env GOPATH)/bin
    - echo "$KUBECONFIG" | base64 -d > ~/.kube/config
  script:
    - hydrophone run --skip "\[Disruptive\]" --wait
    - |
      if [ ! -f "e2e.log" ] || [ ! -f "junit_01.xml" ]; then
        echo "❌ 결과 파일 생성 실패"
        exit 1
      fi
    - fail_count=$(grep -ic "fail" e2e.log || echo "0")
    - |
      if [ "$fail_count" -gt 0 ]; then
        echo "❌ Conformance 검증 실패"
        exit 1
      fi
    - mkdir -p artifacts
    - cp e2e.log junit_01.xml artifacts/
  after_script:
    - kubectl delete namespace conformance --ignore-not-found=true || true
  artifacts:
    paths:
      - artifacts/
    expire_in: 90 days
  only:
    - main
    - schedules
```

## Jenkins Pipeline 예시

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    environment {
        GO_VERSION = '1.21'
    }
    
    stages {
        stage('Setup') {
            steps {
                sh '''
                    # Go 설치 (또는 기존 Go 사용)
                    go version
                    go install sigs.k8s.io/hydrophone@latest
                    export PATH=$PATH:$(go env GOPATH)/bin
                    hydrophone version
                '''
            }
        }
        
        stage('Configure kubectl') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh 'cp $KUBECONFIG ~/.kube/config'
                }
            }
        }
        
        stage('Run Hydrophone') {
            steps {
                sh 'hydrophone run --skip "\[Disruptive\]" --wait'
            }
        }
        
        stage('Check Results') {
            steps {
                script {
                    def resultsExist = sh(
                        script: '[ -f "e2e.log" ] && [ -f "junit_01.xml" ]',
                        returnStatus: true
                    ) == 0
                    
                    if (!resultsExist) {
                        error('❌ 결과 파일 생성 실패')
                    }
                    
                    def failCount = sh(
                        script: 'grep -ic "fail" e2e.log || echo "0"',
                        returnStdout: true
                    ).trim().toInteger()
                    
                    echo "실패 항목: $failCount"
                    
                    if (failCount > 0) {
                        error('❌ Conformance 검증 실패: 배포 차단')
                    }
                }
            }
        }
        
        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: 'e2e.log,junit_01.xml', fingerprint: true
            }
        }
        
        stage('Cleanup') {
            steps {
                sh 'kubectl delete namespace conformance --ignore-not-found=true || true'
            }
        }
    }
    
    post {
        always {
            sh 'kubectl delete namespace conformance --ignore-not-found=true || true'
        }
    }
}
```

## 자동화 베스트 프랙티스

### 1. Gate 역할

```bash
# 실패 시 배포 차단
if [ ! -f "e2e.log" ] || [ ! -f "junit_01.xml" ]; then
  echo "❌ 결과 파일 생성 실패: 배포 차단"
  exit 1
fi

fail_count=$(grep -ic "fail" e2e.log || echo "0")

if [ "$fail_count" -gt 0 ]; then
  echo "❌ Hydrophone 검증 실패: 배포 차단"
  exit 1
fi
```

### 2. 리포트 저장

```bash
# 아티팩트 저장소에 저장
mkdir -p artifacts
timestamp=$(date +%Y%m%d-%H%M%S)
cp e2e.log junit_01.xml artifacts/hydrophone-$timestamp/

# 메타데이터 포함
cat > artifacts/hydrophone-$timestamp/metadata.json << EOF
{
  "timestamp": "$(date -Iseconds)",
  "k8s_version": "$(kubectl version -o json | jq -r '.serverVersion.gitVersion')",
  "hydrophone_version": "$(hydrophone version | head -1)"
}
EOF
```

### 3. 알림 설정

```yaml
# GitHub Actions 예시
- name: Notify on Failure
  if: failure()
  uses: 8398a7/action-slack@v3
  with:
    status: custom
    custom_payload: |
      {
        text: "❌ Hydrophone Conformance 검증 실패",
        attachments: [{
          color: 'danger',
          text: "워크플로우: ${{ github.workflow }}\n커밋: ${{ github.sha }}"
        }]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### 4. 정기 실행

```yaml
# GitHub Actions 스케줄 예시
on:
  schedule:
    - cron: '0 2 * * 0'  # 매주 일요일 새벽 2시
    - cron: '0 2 1 * *'  # 매월 1일 새벽 2시
```

## 관련 문서

- [06-CAT-활용.md](./06-CAT-활용.md) - CAT 관점에서의 Hydrophone 활용
- [08-실무-체크리스트.md](./08-실무-체크리스트.md) - 실무 적용 시 확인 사항
