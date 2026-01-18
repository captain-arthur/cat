# CI/CD 통합 가이드

kube-bench를 CI/CD 파이프라인에 통합하여 자동화된 클러스터 보안 검증을 수행하는 방법을 설명합니다.

## CI/CD 통합 개요

### 목적

- **자동화**: 클러스터 배포 시 자동으로 보안 검증 수행
- **Gate 역할**: 검증 실패 시 배포 차단
- **리포트 관리**: 결과를 자동으로 저장 및 추적

### 통합 위치

```
CI/CD 파이프라인 흐름:
1. 코드 빌드
2. 이미지 빌드
3. 클러스터 배포
4. Sonobuoy Conformance 검증 (CAT 1단계)
5. kube-bench 보안 검증 ← 여기 (CAT 2단계)
6. 성능/수용량 테스트 (CAT 3단계)
7. 복원력 테스트 (CAT 4단계)
8. 운영 승인
```

## GitHub Actions 예시

```yaml
# .github/workflows/cat-kube-bench.yml
name: CAT - kube-bench Security Test

on:
  workflow_dispatch:
  schedule:
    - cron: '0 3 * * 0'  # 주 1회 (일요일 새벽 3시)

jobs:
  security-scan:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Install kube-bench
        run: |
          curl -L https://github.com/aquasecurity/kube-bench/releases/latest/download/kube-bench_0.6.15_linux_amd64.tar.gz -o kube-bench.tar.gz
          tar -xvf kube-bench.tar.gz
          sudo mv kube-bench /usr/local/bin/

      - name: Configure kubectl
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG }}
        run: |
          echo "$KUBECONFIG" | base64 -d > ~/.kube/config

      - name: Run CIS Benchmark Scan
        run: |
          kube-bench run --targets master --json --output master-results.json
          kube-bench run --targets node --json --output node-results.json

      - name: Check Results
        run: |
          FAIL_COUNT=$(cat master-results.json | jq '[.Controls[] | select(.result == "FAIL")] | length')
          echo "FAIL 항목 수: $FAIL_COUNT"
          if [ "$FAIL_COUNT" -gt 0 ]; then
            echo "❌ 보안 검증 실패"
            exit 1
          fi
          echo "✅ 보안 검증 성공"

      - name: Upload Results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: kube-bench-results
          paths:
            - master-results.json
            - node-results.json
```

## GitLab CI 예시

```yaml
# .gitlab-ci.yml
security-scan:
  stage: security
  image: aquasec/kube-bench:latest
  
  before_script:
    - echo "$KUBECONFIG" | base64 -d > ~/.kube/config
  
  script:
    - kube-bench run --targets master --json --output master-results.json
    - |
      FAIL_COUNT=$(cat master-results.json | jq '[.Controls[] | select(.result == "FAIL")] | length')
      if [ "$FAIL_COUNT" -gt 0 ]; then
        exit 1
      fi
  
  artifacts:
    paths:
      - master-results.json
```

## Kubernetes Job 예시

**job.yaml:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench-master
spec:
  template:
    spec:
      hostPID: true
      containers:
      - name: kube-bench
        image: aquasec/kube-bench:latest
        command: ["kube-bench", "run", "--targets", "master", "--json", "--output", "/tmp/results.json"]
        volumeMounts:
        - name: var-lib-etcd
          mountPath: /var/lib/etcd
          readOnly: true
        - name: etc-systemd
          mountPath: /etc/systemd
          readOnly: true
      volumes:
      - name: var-lib-etcd
        hostPath:
          path: /var/lib/etcd
      - name: etc-systemd
        hostPath:
          path: /etc/systemd
      restartPolicy: Never
```

## 관련 문서

- [06-CAT-활용.md](./06-CAT-활용.md) - CAT 관점에서의 kube-bench 활용
- [08-실무-체크리스트.md](./08-실무-체크리스트.md) - 실무 적용 시 확인 사항
