# Kubernetes 통합

Chaos Toolkit을 Kubernetes 환경에서 활용하는 방법을 설명합니다.

## Kubernetes Operator

Kubernetes 클러스터 내에서 실험을 자동 실행하는 Operator입니다.

### Operator 설치

```bash
# Chaos Mesh Operator 설치 (예시)
kubectl apply -f https://raw.githubusercontent.com/chaos-mesh/chaos-mesh/master/manifests/crd.yaml

# 또는 Chaos Toolkit Operator (선택적)
# kubectl apply -f https://raw.githubusercontent.com/chaostoolkit/chaostoolkit-kubernetes-operator/master/deploy/operator.yaml
```

### Experiment CRD 정의

Experiment를 Kubernetes CRD로 정의합니다:

```yaml
apiVersion: chaos.chaostoolkit.org/v1alpha1
kind: Experiment
metadata:
  name: pod-resilience-test
  namespace: chaos-testing
spec:
  experiment:
    version: "1.0.0"
    title: "Pod resilience test"
    steady-state-hypothesis:
      title: "정상 상태 확인"
      probes:
      - type: probe
        name: pod-exists
        provider:
          type: python
          module: chaosk8s.pod.probes
          func: read_pods
          arguments:
            label_selector: "app=test-app"
            ns: "default"
    method:
    - type: action
      name: delete-pod
      provider:
        type: python
        module: chaosk8s.pod.actions
        func: delete_pods
        arguments:
          label_selector: "app=test-app"
          ns: "default"
```

### Experiment 실행

```bash
# Experiment 생성
kubectl apply -f experiment.yaml

# Experiment 상태 확인
kubectl get experiments -n chaos-testing

# Experiment 로그 확인
kubectl logs -n chaos-testing -l app=chaos-operator
```

### Experiment 정리

```bash
# Experiment 삭제
kubectl delete experiment pod-resilience-test -n chaos-testing

# 자동 정리 (Operator가 생성한 리소스 자동 삭제)
kubectl delete experiment pod-resilience-test -n chaos-testing --cascade=true
```

## kubectl Extension

kubectl에서 직접 실험을 실행하는 확장 방법입니다.

### Extension 설치

```bash
# Chaos Toolkit kubectl plugin 설치
kubectl krew install chaos

# 설치 확인
kubectl chaos --help
```

### 실험 실행

```bash
# kubectl을 통한 실험 실행
kubectl chaos run -f experiment.json

# Experiment CRD 기반 실행
kubectl chaos run -f experiment.yaml

# 실험 상태 확인
kubectl chaos status

# 결과 확인
kubectl chaos results
```

## RBAC 설정

Kubernetes에서 실험을 실행하려면 적절한 권한이 필요합니다.

### ServiceAccount 생성

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: chaos-testing
  namespace: chaos-testing
```

### Role/RoleBinding 생성

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: chaos-testing
  namespace: chaos-testing
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: chaos-testing
  namespace: chaos-testing
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: chaos-testing
subjects:
- kind: ServiceAccount
  name: chaos-testing
  namespace: chaos-testing
```

### ClusterRole/ClusterRoleBinding (필요 시)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: chaos-testing
rules:
- apiGroups: [""]
  resources: ["pods", "nodes"]
  verbs: ["get", "list", "watch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: chaos-testing
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: chaos-testing
subjects:
- kind: ServiceAccount
  name: chaos-testing
  namespace: chaos-testing
```

## 네임스페이스 격리

실험을 격리된 네임스페이스에서 실행합니다.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: chaos-testing
  labels:
    chaos-testing: "true"
---
# NetworkPolicy로 격리
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: chaos-testing-isolation
  namespace: chaos-testing
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: chaos-testing
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: chaos-testing
```

## 실전 예시

### 예시: Deployment 롤백 테스트

```yaml
apiVersion: chaos.chaostoolkit.org/v1alpha1
kind: Experiment
metadata:
  name: deployment-rollback-test
  namespace: chaos-testing
spec:
  experiment:
    version: "1.0.0"
    title: "Deployment rollback test"
    description: "Deployment 업데이트 후 롤백이 정상 동작하는지 테스트"
    steady-state-hypothesis:
      title: "정상 상태 확인"
      probes:
      - type: probe
        name: deployment-ready
        provider:
          type: python
          module: chaosk8s.deployment.probes
          func: deployment_is_fully_available
          arguments:
            name: test-app
            ns: "default"
    method:
    - type: action
      name: update-deployment
      provider:
        type: python
        module: chaosk8s.deployment.actions
        func: update_deployment
        arguments:
          name: test-app
          ns: "default"
          spec:
            template:
              spec:
                containers:
                - name: test-app
                  image: test-app:broken
      pauses:
        after: 60
    - type: action
      name: rollback-deployment
      provider:
        type: python
        module: chaosk8s.deployment.actions
        func: rollout_deployment
        arguments:
          name: test-app
          ns: "default"
          rollback_to_revision: 1
      pauses:
        after: 60
```

## 관련 문서

- [04-실험-정의.md](./04-실험-정의.md) - Experiment 파일 작성 방법
- [06-CAT-활용.md](./06-CAT-활용.md) - CAT 관점에서의 활용
- [09-CICD-통합.md](./09-CICD-통합.md) - CI/CD 파이프라인 통합
