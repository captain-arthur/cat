# CI/CD 통합 가이드

Chaos Toolkit을 CI/CD 파이프라인에 통합하여 자동화된 클러스터 복원력 테스트를 수행하는 방법을 설명합니다.

## CI/CD 통합 개요

### 목적

- **자동화**: 클러스터 배포 시 자동으로 복원력 검증 수행
- **Gate 역할**: 검증 실패 시 배포 차단
- **리포트 관리**: 결과를 자동으로 저장 및 추적
- **지속적 검증**: 정기적으로 클러스터 복원력 확인

### 통합 위치

```
CI/CD 파이프라인 흐름:
1. 코드 빌드
2. 이미지 빌드
3. 클러스터 배포
4. Sonobuoy Conformance 검증 (CAT 1단계)
5. kube-bench 보안 검증 (CAT 2단계)
6. 성능/수용량 테스트 (CAT 3단계)
7. Chaos Toolkit 복원력 테스트 ← 여기 (CAT 4단계)
8. 운영 승인
```

## GitHub Actions 예시

### 기본 워크플로우

```yaml
# .github/workflows/cat-chaos-toolkit.yml
name: CAT - Chaos Toolkit Resilience Test

on:
  workflow_dispatch:
    inputs:
      experiment:
        description: 'Experiment 파일 경로'
        required: true
        default: 'experiment.json'
  schedule:
    - cron: '0 3 * * 0'  # 주 1회 (일요일 새벽 3시)
  push:
    branches:
      - main
    paths:
      - 'chaos-toolkit/**'

jobs:
  chaos-resilience:
    runs-on: ubuntu-latest
    timeout-minutes: 30  # 30분 타임아웃
    
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.8'

      - name: Install Chaos Toolkit
        run: |
          pip install -U chaostoolkit chaostoolkit-kubernetes chaostoolkit-reporting

      - name: Configure kubectl
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG }}
        run: |
          echo "$KUBECONFIG" | base64 -d > ~/.kube/config
          kubectl version --short
          kubectl get nodes

      - name: Validate Experiment
        run: |
          EXPERIMENT="${{ github.event.inputs.experiment || 'experiment.json' }}"
          chaos validate $EXPERIMENT

      - name: Run Experiment
        id: run-experiment
        run: |
          EXPERIMENT="${{ github.event.inputs.experiment || 'experiment.json' }}"
          chaos run $EXPERIMENT --journal-path journal.json

      - name: Check Results
        id: check-results
        run: |
          STATUS=$(cat journal.json | jq -r '.status')
          DEVIATED=$(cat journal.json | jq -r '.steady-state-hypothesis.deviated')
          
          echo "status=$STATUS" >> $GITHUB_OUTPUT
          echo "deviated=$DEVIATED" >> $GITHUB_OUTPUT
          
          if [ "$STATUS" != "completed" ] || [ "$DEVIATED" = "true" ]; then
            echo "❌ 복원력 검증 실패"
            echo "Status: $STATUS"
            echo "Deviated: $DEVIATED"
            exit 1
          fi
          
          echo "✅ 복원력 검증 성공"

      - name: Generate Report
        if: always()
        run: |
          chaos report journal.json report.html

      - name: Upload Artifacts
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: chaos-toolkit-results
          path: |
            journal.json
            report.html

      - name: Comment PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const journal = JSON.parse(fs.readFileSync('journal.json', 'utf8'));
            const status = journal.status;
            const deviated = journal['steady-state-hypothesis'].deviated;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Chaos Toolkit 복원력 테스트 결과
              
              **실험 상태:** ${status}
              **Steady-State Hypothesis:** ${deviated ? '❌ 위반' : '✅ 유지'}
              
              ${status === 'completed' && !deviated ? '✅ **합격** - 복원력 검증 성공' : '❌ **불합격** - 복원력 검증 실패'}
              `
            });
```

## GitLab CI 예시

### .gitlab-ci.yml

```yaml
# .gitlab-ci.yml
stages:
  - test
  - chaos-resilience

variables:
  PYTHON_VERSION: "3.8"

chaos-resilience-test:
  stage: chaos-resilience
  image: python:${PYTHON_VERSION}
  
  before_script:
    - pip install -U chaostoolkit chaostoolkit-kubernetes chaostoolkit-reporting
    - echo "$KUBECONFIG" | base64 -d > ~/.kube/config
    - kubectl version --short
  
  script:
    - chaos validate experiment.json
    - chaos run experiment.json --journal-path journal.json
    - |
      STATUS=$(cat journal.json | jq -r '.status')
      DEVIATED=$(cat journal.json | jq -r '.steady-state-hypothesis.deviated')
      
      if [ "$STATUS" != "completed" ] || [ "$DEVIATED" = "true" ]; then
        echo "❌ 복원력 검증 실패"
        exit 1
      fi
      
      echo "✅ 복원력 검증 성공"
    - chaos report journal.json report.html
  
  artifacts:
    when: always
    paths:
      - journal.json
      - report.html
    reports:
      junit: junit.xml  # (JUnit 리포트가 있는 경우)
  
  only:
    - main
    - schedules
```

## Jenkins 예시

### Jenkinsfile

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    environment {
        PYTHON_VERSION = '3.8'
        KUBECONFIG = credentials('kubeconfig')
    }
    
    stages {
        stage('Setup') {
            steps {
                sh '''
                    python3 -m venv venv
                    source venv/bin/activate
                    pip install -U chaostoolkit chaostoolkit-kubernetes chaostoolkit-reporting
                    echo "$KUBECONFIG" | base64 -d > ~/.kube/config
                '''
            }
        }
        
        stage('Validate Experiment') {
            steps {
                sh '''
                    source venv/bin/activate
                    chaos validate experiment.json
                '''
            }
        }
        
        stage('Run Experiment') {
            steps {
                sh '''
                    source venv/bin/activate
                    chaos run experiment.json --journal-path journal.json
                '''
            }
        }
        
        stage('Check Results') {
            steps {
                sh '''
                    source venv/bin/activate
                    STATUS=$(cat journal.json | jq -r '.status')
                    DEVIATED=$(cat journal.json | jq -r '.steady-state-hypothesis.deviated')
                    
                    if [ "$STATUS" != "completed" ] || [ "$DEVIATED" = "true" ]; then
                        echo "❌ 복원력 검증 실패"
                        exit 1
                    fi
                    
                    echo "✅ 복원력 검증 성공"
                '''
            }
        }
        
        stage('Generate Report') {
            steps {
                sh '''
                    source venv/bin/activate
                    chaos report journal.json report.html
                '''
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: 'journal.json,report.html', fingerprint: true
        }
        failure {
            emailext (
                subject: "Chaos Toolkit 복원력 테스트 실패",
                body: "복원력 검증이 실패했습니다. 자세한 내용은 Jenkins 로그를 확인하세요.",
                to: "${env.CHANGE_AUTHOR_EMAIL}"
            )
        }
    }
}
```

## ArgoCD 예시

### Application with PostSync Hook

```yaml
# argo-chaos-toolkit-hook.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: chaos-toolkit-resilience-test
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/chaos-toolkit-experiments
    targetRevision: HEAD
    path: experiments
  destination:
    server: https://kubernetes.default.svc
    namespace: chaos-testing
  syncPolicy:
    hooks:
      postSync:
        - name: chaos-toolkit-test
          apiVersion: batch/v1
          kind: Job
          metadata:
            name: chaos-toolkit-test
            namespace: chaos-testing
          spec:
            template:
              spec:
                serviceAccountName: chaos-testing
                containers:
                - name: chaos-toolkit
                  image: python:3.8
                  command:
                  - /bin/sh
                  - -c
                  - |
                    pip install -U chaostoolkit chaostoolkit-kubernetes
                    chaos run /experiments/pod-resilience.json --journal-path journal.json
                    STATUS=$(cat journal.json | jq -r '.status')
                    DEVIATED=$(cat journal.json | jq -r '.steady-state-hypothesis.deviated')
                    if [ "$STATUS" != "completed" ] || [ "$DEVIATED" = "true" ]; then
                      echo "❌ 복원력 검증 실패"
                      exit 1
                    fi
                  volumeMounts:
                  - name: experiments
                    mountPath: /experiments
                volumes:
                - name: experiments
                  configMap:
                    name: chaos-toolkit-experiments
                restartPolicy: Never
```

## 실전 예시: 전체 CAT 파이프라인

### GitHub Actions 통합

```yaml
# .github/workflows/cat-complete.yml
name: CAT - Complete Pipeline

on:
  workflow_dispatch:
  schedule:
    - cron: '0 2 * * 0'  # 주 1회

jobs:
  cat-complete:
    runs-on: ubuntu-latest
    steps:
      # ... Sonobuoy, kube-bench, kube-burner 단계 ...
      
      - name: CAT 4 - Chaos Toolkit Resilience Test
        run: |
          pip install -U chaostoolkit chaostoolkit-kubernetes
          chaos run chaos-toolkit/pod-resilience.json --journal-path journal-chaos.json
          
          STATUS=$(cat journal-chaos.json | jq -r '.status')
          DEVIATED=$(cat journal-chaos.json | jq -r '.steady-state-hypothesis.deviated')
          
          if [ "$STATUS" != "completed" ] || [ "$DEVIATED" = "true" ]; then
            echo "❌ CAT 4단계 실패: 복원력 검증 실패"
            exit 1
          fi
          
          echo "✅ CAT 4단계 통과: 복원력 검증 성공"
```

## 관련 문서

- [06-CAT-활용.md](./06-CAT-활용.md) - CAT 관점에서의 Chaos Toolkit 활용
- [08-실무-체크리스트.md](./08-실무-체크리스트.md) - 실무 적용 시 확인 사항
