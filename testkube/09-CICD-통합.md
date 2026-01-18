# CI/CD 통합 가이드

Testkube를 CI/CD 파이프라인에 통합하여 자동화된 테스트를 수행하는 방법을 설명합니다.

## CI/CD 통합 개요

### 목적

- **자동화**: 클러스터 배포 시 자동으로 테스트 실행
- **Gate 역할**: 테스트 실패 시 배포 차단
- **리포트 관리**: 결과를 자동으로 저장 및 추적

### 통합 위치

```
CI/CD 파이프라인 흐름:
1. 코드 빌드
2. 이미지 빌드
3. 클러스터 배포
4. Sonobuoy Conformance 검증 (CAT 1단계)
5. 보안 검증 (CAT 2단계)
6. 성능/수용량 테스트 (CAT 3단계)
7. 복원력 테스트 (CAT 4단계)
8. Testkube 테스트 자동화 ← 여기 (CAT 5단계)
9. 운영 승인
```

## GitHub Actions 예시

```yaml
# .github/workflows/cat-testkube.yml
name: CAT - Testkube Test Automation

on:
  workflow_dispatch:
  schedule:
    - cron: '0 3 * * 0'  # 주 1회 (일요일 새벽 3시)

jobs:
  test-automation:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Install Testkube CLI
        run: |
          curl -s https://raw.githubusercontent.com/kubeshop/testkube/main/scripts/install-cli.sh | bash

      - name: Configure kubectl
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG }}
        run: |
          echo "$KUBECONFIG" | base64 -d > ~/.kube/config

      - name: Run Test
        run: |
          testkube run test my-api-test

      - name: Check Results
        run: |
          EXECUTION_ID=$(testkube get execution my-api-test --output json | jq -r '.results[0].id')
          STATUS=$(testkube get execution $EXECUTION_ID --output json | jq -r '.result.status')
          if [ "$STATUS" != "passed" ]; then
            echo "❌ 테스트 자동화 검증 실패"
            exit 1
          fi
          echo "✅ 테스트 자동화 검증 성공"

      - name: Download Artifacts
        if: always()
        run: |
          EXECUTION_ID=$(testkube get execution my-api-test --output json | jq -r '.results[0].id')
          testkube download artifacts $EXECUTION_ID --output-dir ./artifacts
```

## GitLab CI 예시

```yaml
# .gitlab-ci.yml
test-automation:
  stage: test
  image: kubeshop/testkube-cli:latest
  
  before_script:
    - echo "$KUBECONFIG" | base64 -d > ~/.kube/config
  
  script:
    - testkube run test my-api-test
    - |
      EXECUTION_ID=$(testkube get execution my-api-test --output json | jq -r '.results[0].id')
      STATUS=$(testkube get execution $EXECUTION_ID --output json | jq -r '.result.status')
      if [ "$STATUS" != "passed" ]; then
        exit 1
      fi
  
  artifacts:
    paths:
      - artifacts/
```

## 관련 문서

- [06-CAT-활용.md](./06-CAT-활용.md) - CAT 관점에서의 Testkube 활용
- [08-실무-체크리스트.md](./08-실무-체크리스트.md) - 실무 적용 시 확인 사항
