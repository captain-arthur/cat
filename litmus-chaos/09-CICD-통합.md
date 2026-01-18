# CI/CD 통합 가이드

LitmusChaos를 CI/CD 파이프라인에 통합하여 자동화된 클러스터 복원력 테스트를 수행하는 방법을 설명합니다.

## CI/CD 통합 개요

### 목적

- **자동화**: 클러스터 배포 시 자동으로 복원력 검증 수행
- **Gate 역할**: 검증 실패 시 배포 차단
- **리포트 관리**: 결과를 자동으로 저장 및 추적

### 통합 위치

```
CI/CD 파이프라인 흐름:
1. 코드 빌드
2. 이미지 빌드
3. 클러스터 배포
4. Sonobuoy Conformance 검증 (CAT 1단계)
5. kube-bench 보안 검증 (CAT 2단계)
6. 성능/수용량 테스트 (CAT 3단계)
7. LitmusChaos 복원력 테스트 ← 여기 (CAT 4단계)
8. 운영 승인
```

## GitHub Actions 예시

```yaml
# .github/workflows/cat-litmus-chaos.yml
name: CAT - LitmusChaos Resilience Test

on:
  workflow_dispatch:
  schedule:
    - cron: '0 3 * * 0'

jobs:
  chaos-resilience:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Configure kubectl
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG }}
        run: |
          echo "$KUBECONFIG" | base64 -d > ~/.kube/config

      - name: Run Experiment
        run: |
          kubectl apply -f litmus-chaos/pod-delete.yaml

      - name: Wait for Experiment
        run: |
          kubectl wait --for=condition=Ready chaosengine/pod-delete-example -n default --timeout=5m

      - name: Check Results
        run: |
          VERDICT=$(kubectl get chaosresult pod-delete-example-pod-delete -n default -o jsonpath='{.status.experimentStatus.verdict}')
          if [ "$VERDICT" != "Pass" ]; then
            echo "❌ 복원력 검증 실패"
            exit 1
          fi
          echo "✅ 복원력 검증 성공"

      - name: Cleanup
        if: always()
        run: |
          kubectl delete chaosengine pod-delete-example -n default --ignore-not-found=true
```

## 관련 문서

- [06-CAT-활용.md](./06-CAT-활용.md) - CAT 관점에서의 LitmusChaos 활용
- [08-실무-체크리스트.md](./08-실무-체크리스트.md) - 실무 적용 시 확인 사항
