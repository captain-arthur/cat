# CI/CD 통합 가이드

Sonobuoy를 CI/CD 파이프라인에 통합하여 자동화된 클러스터 인수 테스트를 수행하는 방법을 설명합니다.

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
4. Sonobuoy Conformance 검증 ← 여기 (CAT 1단계)
5. kube-bench 보안 검증 (CAT 2단계)
6. 성능/수용량 테스트 (CAT 3단계)
7. 복원력 테스트 (CAT 4단계)
8. 운영 승인
```

## GitHub Actions 예시

### 기본 워크플로우

```yaml
# .github/workflows/cat-sonobuoy.yml
name: CAT - Sonobuoy Conformance Test

on:
  workflow_dispatch:
    inputs:
      mode:
        description: 'Sonobuoy 실행 모드'
        required: true
        default: 'non-disruptive-conformance'
        type: choice
        options:
          - quick
          - non-disruptive-conformance
          - certified-conformance
  schedule:
    - cron: '0 2 * * 0'  # 주 1회 (일요일 새벽 2시)
  push:
    branches:
      - main
    paths:
      - 'cluster/**'

jobs:
  sonobuoy-conformance:
    runs-on: ubuntu-latest
    timeout-minutes: 180  # 3시간 타임아웃
    
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Sonobuoy
        run: |
          VERSION=v0.57.3
          wget https://github.com/vmware-tanzu/sonobuoy/releases/download/${VERSION}/sonobuoy_${VERSION}_linux_amd64.tar.gz
          tar xzf sonobuoy_${VERSION}_linux_amd64.tar.gz
          sudo mv sonobuoy /usr/local/bin/
          chmod +x /usr/local/bin/sonobuoy
          sonobuoy version

      - name: Configure kubectl
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG }}
        run: |
          echo "$KUBECONFIG" | base64 -d > ~/.kube/config
          kubectl version --short
          kubectl get nodes

      - name: Run Sonobuoy
        env:
          MODE: ${{ github.event.inputs.mode || 'non-disruptive-conformance' }}
        run: |
          echo "Sonobuoy 실행 모드: $MODE"
          
          # 타임아웃 설정 (모드별)
          case "$MODE" in
            quick)
              TIMEOUT=600  # 10분
              ;;
            non-disruptive-conformance)
              TIMEOUT=7200  # 2시간
              ;;
            certified-conformance)
              TIMEOUT=14400  # 4시간
              ;;
          esac
          
          sonobuoy run --mode $MODE --wait --timeout $TIMEOUT

      - name: Retrieve Results
        id: retrieve
        run: |
          results=$(sonobuoy retrieve)
          echo "results_file=$results" >> $GITHUB_OUTPUT
          echo "결과 파일: $results"

      - name: Check Results
        id: check
        run: |
          sonobuoy results ${{ steps.retrieve.outputs.results_file }}
          
          # 실패 항목 확인
          if sonobuoy results ${{ steps.retrieve.outputs.results_file }} | grep -q "Failed: [1-9]"; then
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
          cp ${{ steps.retrieve.outputs.results_file }} reports/sonobuoy-results.tar.gz
          
          # 요약 리포트 생성
          cat > reports/summary.md << EOF
          # Sonobuoy Conformance Test 결과 요약
          
          **실행 일시:** $(date '+%Y-%m-%d %H:%M:%S %Z')
          **실행 모드:** ${{ github.event.inputs.mode || 'non-disruptive-conformance' }}
          **워크플로우:** ${{ github.workflow }}
          **커밋:** ${{ github.sha }}
          
          ## 전체 결과
          
          \`\`\`
          $(sonobuoy results ${{ steps.retrieve.outputs.results_file }})
          \`\`\`
          
          ## 합격 여부
          
          $([ "${{ steps.check.outputs.conformance_passed }}" == "true" ] && echo "✅ **합격**" || echo "❌ **불합격**")
          
          EOF
          
          # 결과 요약 출력
          cat reports/summary.md

      - name: Upload Artifacts
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: sonobuoy-results-${{ github.run_number }}
          path: |
            reports/
          retention-days: 90

      - name: Cleanup
        if: always()
        run: |
          sonobuoy delete --wait

      - name: Gate Check
        if: steps.check.outputs.conformance_passed != 'true'
        run: |
          echo "❌ Sonobuoy Conformance 검증 실패: 배포 차단"
          exit 1
```

### 간단한 버전

```yaml
# .github/workflows/sonobuoy-simple.yml
name: Sonobuoy Quick Check

on:
  workflow_dispatch

jobs:
  sonobuoy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Sonobuoy
        run: |
          wget https://github.com/vmware-tanzu/sonobuoy/releases/download/v0.57.3/sonobuoy_0.57.3_linux_amd64.tar.gz
          tar xzf sonobuoy_0.57.3_linux_amd64.tar.gz
          sudo mv sonobuoy /usr/local/bin/
      
      - name: Configure kubectl
        run: echo "${{ secrets.KUBECONFIG }}" | base64 -d > ~/.kube/config
      
      - name: Run Sonobuoy
        run: sonobuoy run --mode non-disruptive-conformance --wait --timeout 7200
      
      - name: Check Results
        run: |
          results=$(sonobuoy retrieve)
          sonobuoy results $results
          if sonobuoy results $results | grep -q "Failed: [1-9]"; then
            echo "❌ Conformance 검증 실패"
            exit 1
          fi
      
      - name: Cleanup
        if: always()
        run: sonobuoy delete --wait
```

## GitLab CI 예시

```yaml
# .gitlab-ci.yml
stages:
  - deploy
  - cat

cat-sonobuoy:
  stage: cat
  image: bitnami/kubectl:latest
  before_script:
    - apt-get update && apt-get install -y wget tar
    - |
      wget https://github.com/vmware-tanzu/sonobuoy/releases/download/v0.57.3/sonobuoy_0.57.3_linux_amd64.tar.gz
      tar xzf sonobuoy_0.57.3_linux_amd64.tar.gz
      chmod +x sonobuoy
      mv sonobuoy /usr/local/bin/
    - echo "$KUBECONFIG" | base64 -d > ~/.kube/config
  script:
    - sonobuoy run --mode non-disruptive-conformance --wait --timeout 7200
    - results=$(sonobuoy retrieve)
    - sonobuoy results $results
    - |
      if sonobuoy results $results | grep -q "Failed: [1-9]"; then
        echo "❌ Conformance 검증 실패"
        exit 1
      fi
    - mkdir -p artifacts
    - mv $results artifacts/sonobuoy-results.tar.gz
  after_script:
    - sonobuoy delete --wait
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
        SONOBUOY_VERSION = 'v0.57.3'
    }
    
    stages {
        stage('Setup') {
            steps {
                sh '''
                    wget https://github.com/vmware-tanzu/sonobuoy/releases/download/${SONOBUOY_VERSION}/sonobuoy_${SONOBUOY_VERSION}_linux_amd64.tar.gz
                    tar xzf sonobuoy_${SONOBUOY_VERSION}_linux_amd64.tar.gz
                    sudo mv sonobuoy /usr/local/bin/
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
        
        stage('Run Sonobuoy') {
            steps {
                sh 'sonobuoy run --mode non-disruptive-conformance --wait --timeout 7200'
            }
        }
        
        stage('Check Results') {
            steps {
                script {
                    def results = sh(script: 'sonobuoy retrieve', returnStdout: true).trim()
                    def resultSummary = sh(script: "sonobuoy results ${results}", returnStdout: true)
                    
                    echo resultSummary
                    
                    if (resultSummary.contains('Failed: [1-9]')) {
                        error('❌ Conformance 검증 실패: 배포 차단')
                    }
                    
                    archiveArtifacts artifacts: "${results}", fingerprint: true
                }
            }
        }
        
        stage('Cleanup') {
            steps {
                sh 'sonobuoy delete --wait'
            }
        }
    }
    
    post {
        always {
            sh 'sonobuoy delete --wait || true'
        }
    }
}
```

## ArgoCD 통합 예시

### ArgoCD Application with PostSync Hook

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-cluster
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/cluster-config
    targetRevision: main
    path: clusters/my-cluster
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    automated:
      prune: true
      selfHeal: true
    hooks:
      - type: PostSync
        metadata:
          name: sonobuoy-conformance
        spec:
          template:
            apiVersion: batch/v1
            kind: Job
            metadata:
              name: sonobuoy-conformance
              namespace: sonobuoy
            spec:
              backoffLimit: 1
              template:
                spec:
                  serviceAccountName: sonobuoy
                  containers:
                    - name: sonobuoy
                      image: docker.io/sonobuoy/sonobuoy:v0.57.3
                      command:
                        - /bin/sh
                        - -c
                        - |
                          sonobuoy run --mode non-disruptive-conformance --wait --timeout 7200
                          results=$(sonobuoy retrieve)
                          sonobuoy results $results
                          if sonobuoy results $results | grep -q "Failed: [1-9]"; then
                            echo "❌ Conformance 검증 실패"
                            exit 1
                          fi
                          echo "✅ Conformance 검증 성공"
                  restartPolicy: Never
```

## 자동화 베스트 프랙티스

### 1. Gate 역할

```bash
# 실패 시 배포 차단
if ! sonobuoy results $results | grep -q "Failed: 0"; then
  echo "❌ Sonobuoy 검증 실패: 배포 차단"
  exit 1
fi
```

### 2. 리포트 저장

```bash
# 아티팩트 저장소에 저장
mkdir -p artifacts
mv $results artifacts/sonobuoy-$(date +%Y%m%d-%H%M%S).tar.gz

# 메타데이터 포함
cat > artifacts/metadata.json << EOF
{
  "timestamp": "$(date -Iseconds)",
  "mode": "non-disruptive-conformance",
  "k8s_version": "$(kubectl version -o json | jq -r '.serverVersion.gitVersion')",
  "sonobuoy_version": "$(sonobuoy version --short)"
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
        text: "❌ Sonobuoy Conformance 검증 실패",
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

- [06-CAT-활용.md](./06-CAT-활용.md) - CAT 관점에서의 Sonobuoy 활용
- [08-실무-체크리스트.md](./08-실무-체크리스트.md) - 실무 적용 시 확인 사항
