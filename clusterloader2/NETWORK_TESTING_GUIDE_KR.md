# ClusterLoader2 네트워크 부하 테스트 완벽 가이드

> Kubernetes 클러스터 인수 테스트를 위한 네트워크 성능 측정

---

## 목차

1. [들어가기 전에: 기본 개념](#1-들어가기-전에-기본-개념)
2. [테스트 시나리오 1: 네트워크 성능 테스트](#2-테스트-시나리오-1-네트워크-성능-테스트)
3. [테스트 시나리오 2: L4 External LoadBalancer 테스트](#3-테스트-시나리오-2-l4-external-loadbalancer-테스트)
4. [테스트 시나리오 3: L4 Internal LoadBalancer 테스트](#4-테스트-시나리오-3-l4-internal-loadbalancer-테스트)
5. [테스트 시나리오 4: NetworkPolicy 테스트](#5-테스트-시나리오-4-networkpolicy-테스트)
6. [실행 방법과 결과 해석](#6-실행-방법과-결과-해석)

---

## 1. 들어가기 전에: 기본 개념

### 1.1 ClusterLoader2 테스트는 어떻게 구성되나?

ClusterLoader2의 모든 테스트는 **YAML 파일**로 정의됩니다. 크게 두 가지로 나뉩니다:

```
┌─────────────────────────────────────────────────────────────────┐
│                        테스트 구성요소                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   config.yaml (테스트 설정)          리소스 템플릿               │
│   ┌───────────────────────┐         ┌──────────────────────┐   │
│   │ steps:                │         │ service.yaml         │   │
│   │   - phases:           │ ──────► │ dep.yaml             │   │
│   │       (리소스 생성)    │  참조   │ pod.yaml             │   │
│   │   - measurements:     │         │ ...                  │   │
│   │       (측정 시작/수집) │         └──────────────────────┘   │
│   └───────────────────────┘                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Step, Phase, Measurement란?

테스트는 **Step(단계)** 들의 연속입니다. 각 Step에는 두 종류가 있습니다:

| 구분 | 하는 일 | 예시 |
|------|---------|------|
| **Phase** | Kubernetes 리소스를 생성/수정/삭제 | Pod 100개 생성, Service 10개 생성 |
| **Measurement** | 성능 데이터를 측정 | 지연시간 기록 시작, 결과 수집 |

**중요**: Step 안의 Phase들은 **병렬**로 실행되고, Step들은 **순차적**으로 실행됩니다.

```yaml
# 예시: config.yaml의 구조
steps:
- name: "Step 1 - 측정 시작"        # Step 1
  measurements:                      # ← Measurement
    - Method: ServiceCreationLatency
      Params:
        action: start

- name: "Step 2 - 리소스 생성"       # Step 2 (Step 1 완료 후 실행)
  phases:                            # ← Phase
    - replicasPerNamespace: 10
      objectTemplatePath: service.yaml

- name: "Step 3 - 결과 수집"         # Step 3 (Step 2 완료 후 실행)
  measurements:
    - Method: ServiceCreationLatency
      Params:
        action: gather
```

### 1.3 트래픽은 어떻게 발생시키나?

ClusterLoader2는 크게 **두 가지 방식**으로 네트워크 트래픽을 발생시킵니다:

#### 방식 1: 전용 Worker Pod 사용 (네트워크 성능 테스트)

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Worker Pod (Client)              Worker Pod (Server)          │
│   ┌─────────────────┐              ┌─────────────────┐         │
│   │ iperf3 client   │ ──TCP/UDP──► │ iperf3 server   │         │
│   │ siege (HTTP)    │ ──HTTP────► │ HTTP server     │         │
│   └─────────────────┘              └─────────────────┘         │
│                                                                 │
│   이미지: gcr.io/k8s-testimages/netperfbenchmark:0.3           │
│   내장 도구: iperf2, iperf3, siege                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 방식 2: 실제 서비스 접근 (LoadBalancer 테스트)

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Exec Service Pod                 LoadBalancer Service         │
│   ┌─────────────────┐              ┌─────────────────┐         │
│   │                 │ ──curl────►  │  ───────────────│──►nginx │
│   │ curl 명령 실행   │              │   External IP   │         │
│   └─────────────────┘              └─────────────────┘         │
│                                                                 │
│   목적: Service가 실제로 응답하는지 확인                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.4 testing 폴더 구조

```
testing/
├── network/          # 네트워크 성능 테스트 (Pod 간 직접 통신)
│   ├── config.yaml
│   ├── suite.yaml
│   └── *_override.yaml
│
├── l4lb/             # External LoadBalancer 테스트
│   ├── config.yaml
│   ├── service.yaml
│   └── dep.yaml
│
├── l4ilb/            # Internal LoadBalancer 테스트
│   ├── config.yaml
│   ├── modules/
│   │   ├── measurements.yaml
│   │   └── services.yaml
│   ├── service.yaml
│   └── dep.yaml
│
└── load/modules/network-policy/   # NetworkPolicy 테스트
    ├── net-policy-enforcement-latency.yaml
    └── net-policy-metrics.yaml
```

---

## 2. 테스트 시나리오 1: 네트워크 성능 테스트

**경로**: `testing/network/`

### 2.1 이 테스트는 무엇을 측정하나?

> **"Pod와 Pod가 직접 통신할 때 네트워크 성능이 얼마나 나오는가?"**

마이크로서비스 아키텍처에서 Pod들은 끊임없이 서로 통신합니다. 이 테스트는 CNI(Container Network Interface) 플러그인의 실제 성능을 측정합니다.

### 2.2 테스트 파일 구성

```
testing/network/
├── config.yaml                 # 메인 테스트 로직
├── suite.yaml                  # 6가지 테스트 조합 정의
├── tcp_protocol_override.yaml  # CL2_PROTOCOL: TCP
├── udp_protocol_override.yaml  # CL2_PROTOCOL: UDP
├── http_protocol_override.yaml # CL2_PROTOCOL: HTTP
├── 1_1_ratio_override.yaml     # 서버1:클라이언트1
└── 50_50_ratio_override.yaml   # 서버50:클라이언트50
```

### 2.3 config.yaml 한 줄씩 분석

```yaml
# testing/network/config.yaml

# 변수 정의 (override 파일에서 값을 받음)
{{$PROTOCOL := .CL2_PROTOCOL}}              # TCP, UDP, 또는 HTTP
{{$NUMBER_OF_SERVERS := .CL2_NUMBER_OF_SERVERS}}
{{$NUMBER_OF_CLIENTS := .CL2_NUMBER_OF_CLIENTS}}

name: network_performance    # 테스트 이름
namespace:
  number: 1                  # 네임스페이스 1개만 사용

steps:
# ──────────────────────────────────────────────────────────
# Step 1: 측정 시작
# ──────────────────────────────────────────────────────────
- name: Start network performance measurement
  measurements:
    - Identifier: NetworkPerformanceMetrics
      Method: NetworkPerformanceMetrics
      Params:
        action: start           # ← "시작해라"
        duration: 10s           # 10초 동안 트래픽 발생
        protocol: {{$PROTOCOL}} # TCP/UDP/HTTP
        numberOfServers: {{$NUMBER_OF_SERVERS}}
        numberOfClients: {{$NUMBER_OF_CLIENTS}}

# ──────────────────────────────────────────────────────────
# Step 2: 결과 수집
# ──────────────────────────────────────────────────────────
- name: Gather network performance measurement
  measurements:
    - Identifier: NetworkPerformanceMetrics
      Method: NetworkPerformanceMetrics
      Params:
        action: gather          # ← "결과 모아라"
```

### 2.4 action: start가 실행되면 무슨 일이 벌어지나?

**순서대로 설명합니다:**

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: 클러스터 준비                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   1. "netperf" 네임스페이스 생성                                 │
│   2. NetworkTestRequest CRD 등록                                │
│   3. RBAC (ClusterRoleBinding) 생성                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: Worker Pod 배포                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Deployment 생성:                                              │
│   - replicas: numberOfClients + numberOfServers                 │
│   - image: gcr.io/k8s-testimages/netperfbenchmark:0.3          │
│                                                                 │
│   예: 1:1 테스트 → 2개 Pod                                       │
│       50:50 테스트 → 100개 Pod                                   │
│                                                                 │
│   모든 Pod가 Running 상태가 될 때까지 대기 (최대 5분)             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Pod 페어링                                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   핵심: "서로 다른 노드에 있는 Pod끼리 짝을 짓는다"               │
│                                                                 │
│   왜? 같은 노드 내 통신은 실제 네트워크를 안 타기 때문에         │
│       CNI 성능을 정확히 측정하려면 노드 간 통신이 필요함          │
│                                                                 │
│   알고리즘 (Heap 기반):                                          │
│   ┌────────────────────────────────────────────────────────┐    │
│   │ Node A: [Pod1, Pod2, Pod3]  ←── 가장 많은 Pod          │    │
│   │ Node B: [Pod4, Pod5]                                   │    │
│   │ Node C: [Pod6]                                         │    │
│   │                                                        │    │
│   │ 결과:                                                   │    │
│   │   Pair 1: Pod1(A) ↔ Pod4(B)  ← 서로 다른 노드          │    │
│   │   Pair 2: Pod2(A) ↔ Pod5(B)                            │    │
│   │   Pair 3: Pod3(A) ↔ Pod6(C)                            │    │
│   └────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 4: 테스트 요청 생성 (CRD)                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   각 Pod 쌍마다 NetworkTestRequest CR 생성:                      │
│                                                                 │
│   apiVersion: clusterloader.io/v1alpha1                         │
│   kind: NetworkTestRequest                                      │
│   metadata:                                                     │
│     name: network-test-0                                        │
│   spec:                                                         │
│     clientPodName: worker-xxxxx                                 │
│     serverPodName: worker-yyyyy                                 │
│     serverPodIP: 10.0.0.100                                     │
│     protocol: TCP                                               │
│     duration: 10                                                │
│     clientStartTimestamp: 1699999999  ← 모든 Pod가 동시에 시작  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 5: 트래픽 발생 (Worker Pod 내부)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Worker Pod들은 CR을 Watch하고 있다가...                        │
│                                                                 │
│   clientStartTimestamp 시간이 되면:                              │
│                                                                 │
│   ┌─ TCP 프로토콜 ─────────────────────────────────────────┐    │
│   │  Client Pod:                                           │    │
│   │    $ iperf3 -c <serverPodIP> -t 10                     │    │
│   │                                                        │    │
│   │  Server Pod:                                           │    │
│   │    $ iperf3 -s (이미 대기 중)                          │    │
│   └────────────────────────────────────────────────────────┘    │
│                                                                 │
│   ┌─ UDP 프로토콜 ─────────────────────────────────────────┐    │
│   │  Client Pod:                                           │    │
│   │    $ iperf -c <serverPodIP> -u -t 10 -b 750M           │    │
│   │                         └── 750Mbps로 UDP 전송         │    │
│   └────────────────────────────────────────────────────────┘    │
│                                                                 │
│   ┌─ HTTP 프로토콜 ────────────────────────────────────────┐    │
│   │  Client Pod:                                           │    │
│   │    $ siege -t 10s http://<serverPodIP>/                │    │
│   └────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 6: 결과 보고 (CR status 업데이트)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   테스트 완료 후 Worker Pod가 CR의 status를 업데이트:            │
│                                                                 │
│   status:                                                       │
│     metrics: [5234.5, 0.32, 0.001, ...]                        │
│              │        │     └── 추가 메트릭들                    │
│              │        └── Jitter (UDP)                          │
│              └── Bandwidth (TCP) 또는 PPS (UDP)                  │
│     error: ""                                                   │
│     workerDelay: 0.05                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.5 측정되는 메트릭 상세

#### TCP 메트릭

| 메트릭 | 의미 | 단위 | 측정 도구 |
|--------|------|------|-----------|
| **Throughput** (tcpBandwidth) | 초당 전송 데이터량 | kbytes/sec | iperf3 |

```
TCP는 "얼마나 빨리 데이터를 보낼 수 있는가"를 측정합니다.

예: Throughput = 5000 kbytes/sec
    = 초당 5MB 전송 가능
    = 40 Mbps
```

#### UDP 메트릭

| 메트릭 | 의미 | 단위 | 좋은 값 |
|--------|------|------|---------|
| **Jitter** | 패킷 도착 간격의 변동 | ms | < 1ms |
| **Latency** | 평균 지연시간 | ms | < 5ms |
| **Lost Packets** | 손실된 패킷 비율 | % | < 0.1% |
| **Packet Per Second** | 초당 처리 패킷 수 | pps | 높을수록 좋음 |

```
UDP는 실시간 통신 품질을 측정합니다.

┌─────────────────────────────────────────────────────────────┐
│ Jitter란?                                                   │
│                                                             │
│ 이상적:  ──●────●────●────●────●──  (일정한 간격)           │
│                                                             │
│ Jitter:  ──●──●──────●─●──────●──  (불규칙한 간격)          │
│                ↑       ↑                                    │
│            간격이 들쭉날쭉 = Jitter가 높음                   │
│                                                             │
│ VoIP, 게임 등 실시간 서비스에서 Jitter가 높으면 끊김 발생     │
└─────────────────────────────────────────────────────────────┘
```

#### HTTP 메트릭

| 메트릭 | 의미 | 단위 |
|--------|------|------|
| **Response Time** | HTTP 요청-응답 시간 | seconds |

### 2.6 1:1 vs 50:50 결과의 차이

**1:1 테스트** (Baseline):
```json
{
  "Metric": "Throughput",
  "Data": { "Value": 5234.5 },    // 단일 값
  "Unit": "kbytes/sec"
}
```

**50:50 테스트** (Scale):
```json
{
  "Metric": "Throughput",
  "Data": {
    "Perc05": 3800.0,    // 하위 5% (가장 느린 Pod 쌍들)
    "Perc50": 4950.0,    // 중앙값 (일반적인 성능)
    "Perc95": 5400.0     // 상위 95% (가장 빠른 Pod 쌍들)
  },
  "Unit": "kbytes/sec"
}
```

```
왜 Percentile로 보나?

50개 Pod 쌍이 동시에 통신하면 각각 성능이 다릅니다.
- 어떤 쌍은 빠른 노드 간 통신
- 어떤 쌍은 네트워크 혼잡 구간 통과
- 어떤 쌍은 리소스 경쟁으로 느림

Perc05/Perc50 비율이 클수록 성능 편차가 큼 (나쁨)
```

### 2.7 suite.yaml - 6가지 테스트 조합

```yaml
# testing/network/suite.yaml

# TCP 테스트
- identifier: tcp-1:1
  configPath: testing/network/config.yaml
  overridePaths:
    - testing/network/tcp_protocol_override.yaml   # CL2_PROTOCOL: TCP
    - testing/network/1_1_ratio_override.yaml      # 서버1, 클라이언트1

- identifier: tcp-50:50
  configPath: testing/network/config.yaml
  overridePaths:
    - testing/network/tcp_protocol_override.yaml
    - testing/network/50_50_ratio_override.yaml    # 서버50, 클라이언트50

# UDP 테스트 (동일 구조)
- identifier: udp-1:1
- identifier: udp-50:50

# HTTP 테스트 (동일 구조)
- identifier: http-1:1
- identifier: http-50:50
```

---

## 3. 테스트 시나리오 2: L4 External LoadBalancer 테스트

**경로**: `testing/l4lb/`

### 3.1 이 테스트는 무엇을 측정하나?

> **"LoadBalancer 타입 Service를 만들면 실제로 트래픽을 받을 수 있기까지 얼마나 걸리는가?"**

클라우드 환경에서 LoadBalancer는 외부 IP를 할당받고, 백엔드 Pod들을 등록하고, Health Check를 통과해야 합니다. 이 전체 과정의 시간을 측정합니다.

### 3.2 테스트 파일 구성

```
testing/l4lb/
├── config.yaml      # 메인 테스트 로직 (Step 정의)
├── service.yaml     # LoadBalancer Service 템플릿
└── dep.yaml         # Backend Deployment 템플릿 (nginx)
```

### 3.3 config.yaml 전체 흐름

```yaml
# testing/l4lb/config.yaml

# ─────────────────────────────────────────────────────────────────
# 변수 정의
# ─────────────────────────────────────────────────────────────────
{{$LOAD_BALANCER_BACKEND_SIZE := DefaultParam .CL2_LOAD_BALANCER_BACKEND_SIZE 5}}
# └── 각 LB 뒤에 붙을 nginx Pod 개수 (기본 5개)

{{$LOAD_BALANCER_REPLICAS := DefaultParam .CL2_LOAD_BALANCER_REPLICAS 3}}
# └── 생성할 LoadBalancer Service 개수 (기본 3개)

{{$LOAD_BALANCER_TYPE := DefaultParam .CL2_LOAD_BALANCER_TYPE "EXTERNAL"}}
# └── EXTERNAL (외부 IP) 또는 INTERNAL (내부 IP)

{{$lbQPS := 20}}
# └── 초당 20개 객체 생성 속도

name: l4lbload
namespace:
  number: 1

# ─────────────────────────────────────────────────────────────────
# TuningSet: 리소스 생성 속도 제어
# ─────────────────────────────────────────────────────────────────
tuningSets:
- name: LBConstantQPS
  qpsLoad:
    qps: 20   # 초당 20개 객체 생성

steps:
# ═══════════════════════════════════════════════════════════════
# Step 1: 측정 시작
# ═══════════════════════════════════════════════════════════════
- name: Initialize Measurements
  measurements:
  - Identifier: LBServiceCreationLatency
    Method: ServiceCreationLatency
    Params:
      action: start                    # 측정 시작
      labelSelector: test = l4lb-load  # 이 라벨 가진 Service만 추적
      waitTimeout: 30m

  - Identifier: WaitForRunningDeployments
    Method: WaitForControlledPodsRunning
    Params:
      action: start
      labelSelector: test = l4lb-load

# ═══════════════════════════════════════════════════════════════
# Step 2: LB와 백엔드 생성
# ═══════════════════════════════════════════════════════════════
- name: Creating LBs
  phases:
  - namespaceRange:
      min: 1
      max: 1
    replicasPerNamespace: 3           # LB 3개 생성
    tuningSet: LBConstantQPS          # QPS 20으로 생성
    objectBundle:
    - basename: lb-service
      objectTemplatePath: service.yaml
      templateFillMap:
        DeploymentBaseName: lb-dep
        ExternalTrafficPolicy: Cluster
        LoadBalancerType: EXTERNAL
    - basename: lb-dep
      objectTemplatePath: dep.yaml
      templateFillMap:
        NumReplicas: 5                # 각 LB당 nginx 5개

# ═══════════════════════════════════════════════════════════════
# Step 3: LB Ready 대기
# ═══════════════════════════════════════════════════════════════
- name: Wait for LBs to be ready
  measurements:
  - Identifier: LBServiceCreationLatency
    Method: ServiceCreationLatency
    Params:
      action: waitForReady            # 모든 LB가 응답할 때까지 대기

  - Identifier: WaitForRunningDeployments
    Method: WaitForControlledPodsRunning
    Params:
      action: gather

# ═══════════════════════════════════════════════════════════════
# Step 4: NodeSync 지연시간 측정
# ═══════════════════════════════════════════════════════════════
- name: Measure NodeSync latency
  measurements:
  - Identifier: NodeSyncLatency
    Method: LoadBalancerNodeSyncLatency
    Params:
      action: measure
      labelSelector: test = l4lb-load

# ═══════════════════════════════════════════════════════════════
# Step 5: LB 삭제
# ═══════════════════════════════════════════════════════════════
- name: Deleting LBs
  phases:
  - namespaceRange:
      min: 1
      max: 1
    replicasPerNamespace: 0          # 0으로 설정 = 삭제
    objectBundle:
    - basename: lb-service
      objectTemplatePath: service.yaml
    - basename: lb-dep
      objectTemplatePath: dep.yaml

# ═══════════════════════════════════════════════════════════════
# Step 6: 삭제 완료 대기
# ═══════════════════════════════════════════════════════════════
- name: Wait for LBs to be deleted
  measurements:
  - Identifier: LBServiceCreationLatency
    Method: ServiceCreationLatency
    Params:
      action: waitForDeletion

# ═══════════════════════════════════════════════════════════════
# Step 7: 결과 수집
# ═══════════════════════════════════════════════════════════════
- name: Gather Measurements
  measurements:
  - Identifier: LBServiceCreationLatency
    Method: ServiceCreationLatency
    Params:
      action: gather

  - Identifier: NodeSyncLatency
    Method: LoadBalancerNodeSyncLatency
    Params:
      action: gather
```

### 3.4 생성되는 리소스 상세

#### service.yaml - LoadBalancer Service

```yaml
# testing/l4lb/service.yaml

apiVersion: v1
kind: Service
metadata:
  name: {{.Name}}               # lb-service-0, lb-service-1, lb-service-2
  labels:
    test: l4lb-load             # 이 라벨로 측정 대상 식별
  annotations:
    networking.gke.io/load-balancer-type: {{.LoadBalancerType}}
    # └── GKE에서 External/Internal 구분
spec:
  type: LoadBalancer            # 핵심: LoadBalancer 타입
  externalTrafficPolicy: {{.ExternalTrafficPolicy}}
  selector:
    name: {{.DeploymentBaseName}}-{{.Index}}
    # └── lb-dep-0, lb-dep-1, lb-dep-2
  ports:
  - port: 80
    targetPort: 80
```

#### dep.yaml - Backend Deployment

```yaml
# testing/l4lb/dep.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{.Name}}               # lb-dep-0, lb-dep-1, lb-dep-2
  labels:
    test: l4lb-load
spec:
  replicas: {{.NumReplicas}}    # 5개
  selector:
    matchLabels:
      name: {{.Name}}
  template:
    spec:
      containers:
      - name: {{.Name}}
        image: nginx            # 단순 nginx 사용
        ports:
        - containerPort: 80
```

### 3.5 ServiceCreationLatency가 측정하는 것

```
┌─────────────────────────────────────────────────────────────────┐
│                  LoadBalancer 생성 타임라인                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  시간 ──────────────────────────────────────────────────────►   │
│                                                                 │
│       t0              t1                      t2                │
│       │               │                       │                 │
│       ▼               ▼                       ▼                 │
│   ┌───────┐      ┌─────────┐           ┌───────────┐           │
│   │Service│      │External │           │ curl 성공  │           │
│   │ 생성  │      │IP 할당  │           │ (3회 연속) │           │
│   └───────┘      └─────────┘           └───────────┘           │
│                                                                 │
│   Phase:                                                        │
│   creating      ipAssigning            reachability            │
│                                                                 │
│   측정값:                                                        │
│   ├─ create_to_assigned ─┤ (t1-t0)                             │
│   │                      ├─ assigned_to_available ─┤ (t2-t1)   │
│   ├──────── create_to_available ───────────────────┤ (t2-t0)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**각 단계의 의미:**

| 단계 | 무슨 일이 일어나나? | 왜 시간이 걸리나? |
|------|---------------------|-------------------|
| **creating → ipAssigning** | Cloud Provider가 LB 생성, IP 할당 | API 호출, 리소스 프로비저닝 |
| **ipAssigning → reachability** | 백엔드 등록, Health Check 통과 | 네트워크 설정, Health Check 간격 |

### 3.6 도달성(Reachability) 확인 방법

```go
// 실제 코드에서 curl로 확인
func (p *pingChecker) run() {
    success := 0
    for {
        // exec-service Pod에서 curl 실행
        pod, _ := execservice.GetPod()
        address := p.svc.Status.LoadBalancer.Ingress[0].IP

        command := fmt.Sprintf("curl %s:80", address)
        _, err := execservice.RunCommand(ctx, pod, command)

        if err == nil {
            success++
            if success == 3 {  // 3회 연속 성공하면 OK
                // reachability 시간 기록
                return
            }
        } else {
            success = 0       // 실패하면 리셋
            time.Sleep(1 * time.Second)
        }
    }
}
```

### 3.7 NodeSync Latency란?

> **"LB 백엔드 노드 목록이 변경될 때 얼마나 빨리 반영되는가?"**

```
┌─────────────────────────────────────────────────────────────────┐
│                     NodeSync 측정 시나리오                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Before:                                                        │
│  LoadBalancer ──► Node A, Node B, Node C (백엔드 3개)           │
│                                                                 │
│  Action:                                                        │
│  Node A에 레이블 추가:                                           │
│    node.kubernetes.io/exclude-from-external-load-balancers=true │
│                                                                 │
│  After (NodeSync 완료 후):                                      │
│  LoadBalancer ──► Node B, Node C (백엔드 2개)                   │
│                                                                 │
│  측정값:                                                        │
│  레이블 추가 시점 → UpdatedLoadBalancer 이벤트 발생 시점         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. 테스트 시나리오 3: L4 Internal LoadBalancer 테스트

**경로**: `testing/l4ilb/`

### 4.1 External vs Internal LoadBalancer

| 구분 | External LB | Internal LB |
|------|-------------|-------------|
| **IP 타입** | 외부에서 접근 가능한 Public IP | VPC 내부 Private IP |
| **용도** | 인터넷 트래픽 처리 | 내부 마이크로서비스 통신 |
| **GCP Annotation** | `networking.gke.io/load-balancer-type: External` | `cloud.google.com/load-balancer-type: Internal` |

### 4.2 테스트 파일 구성

```
testing/l4ilb/
├── config.yaml              # 메인 테스트 로직
├── modules/
│   ├── measurements.yaml    # 측정 모듈 (재사용)
│   └── services.yaml        # 서비스 생성 모듈
├── service.yaml             # Internal LB 템플릿
└── dep.yaml                 # Backend Deployment 템플릿
```

### 4.3 config.yaml 분석

```yaml
# testing/l4ilb/config.yaml

# 백엔드 크기별 LB 개수
{{$LARGE_BACKEND_LB_SERVICE_COUNT := DefaultParam .CL2_LARGE_BACKEND_LB_SERVICE_COUNT 2}}
{{$MEDIUM_BACKEND_LB_SERVICE_COUNT := DefaultParam .CL2_MEDIUM_BACKEND_LB_SERVICE_COUNT 2}}
{{$SMALL_BACKEND_LB_SERVICE_COUNT := DefaultParam .CL2_SMALL_BACKEND_LB_SERVICE_COUNT 2}}

# 총 6개 ILB 생성 (Large 2 + Medium 2 + Small 2)

name: ilbload
namespace:
  number: 1

tuningSets:
- name: ILBConstantQPS
  qpsLoad:
    qps: 20

steps:
# Step 1: 측정 시작 (모듈 호출)
- module:
    path: /modules/measurements.yaml
    params:
      action: start

# Step 2: Pod 생성 대기 시작
- name: Start measurement for running pods
  measurements:
  - Identifier: WaitForRunningDeployments
    Method: WaitForControlledPodsRunning
    Params:
      action: start
      labelSelector: group = ilb-load

# Step 3: ILB 생성 (모듈 호출)
- module:
    path: /modules/services.yaml
    params:
      actionName: Create
      largeBackendLbServiceCount: 2    # Large 백엔드 LB 2개
      mediumBackendLbServiceCount: 2   # Medium 백엔드 LB 2개
      smallBackendLbServiceCount: 2    # Small 백엔드 LB 2개

# Step 4: Ready 대기 (모듈 호출)
- module:
    path: /modules/measurements.yaml
    params:
      action: waitForReady

# Step 5: Pod 생성 완료 확인
- name: Waiting for objects creation to be completed
  measurements:
  - Identifier: WaitForRunningDeployments
    Method: WaitForControlledPodsRunning
    Params:
      action: gather

# Step 6: ILB 삭제
- module:
    path: /modules/services.yaml
    params:
      actionName: Delete
      largeBackendLbServiceCount: 0
      mediumBackendLbServiceCount: 0
      smallBackendLbServiceCount: 0

# Step 7: 결과 수집 (모듈 호출)
- module:
    path: /modules/measurements.yaml
    params:
      action: gather
```

### 4.4 모듈 상세: measurements.yaml

```yaml
# testing/l4ilb/modules/measurements.yaml

{{$action := .action}}
{{$ilbWaitTimeout := DefaultParam .CL2_ILB_WAIT_TIMEOUT "10m"}}

steps:
- name: Service creation latency measurements - '{{$action}}'
  measurements:
  # Large 백엔드 LB 측정 (별도 추적)
  - Identifier: ServiceCreationLatencyLarge
    Method: ServiceCreationLatency
    Params:
      action: {{$action}}
      waitTimeout: {{$ilbWaitTimeout}}
      labelSelector: size = ilb-large     # 라벨로 구분

  # Medium 백엔드 LB 측정
  - Identifier: ServiceCreationLatencyMedium
    Method: ServiceCreationLatency
    Params:
      action: {{$action}}
      labelSelector: size = ilb-medium

  # Small 백엔드 LB 측정
  - Identifier: ServiceCreationLatencySmall
    Method: ServiceCreationLatency
    Params:
      action: {{$action}}
      labelSelector: size = ilb-small
```

### 4.5 모듈 상세: services.yaml

```yaml
# testing/l4ilb/modules/services.yaml

{{$LARGE_BACKEND_SIZE := DefaultParam .CL2_LARGE_BACKEND_SIZE 300}}
{{$MEDIUM_BACKEND_SIZE := DefaultParam .CL2_MEDIUM_BACKEND_SIZE 150}}
{{$SMALL_BACKEND_SIZE := DefaultParam .CL2_SMALL_BACKEND_SIZE 10}}

steps:
- name: {{$actionName}} ILBs
  phases:
  # Large 백엔드 ILB (300 Pods)
  - replicasPerNamespace: {{$LARGE_BACKEND_LB_SERVICE_COUNT}}
    tuningSet: ILBConstantQPS
    objectBundle:
    - basename: large-backends-service
      objectTemplatePath: service.yaml
      templateFillMap:
        ILBSizeLabel: ilb-large
    - basename: large-backends-dep
      objectTemplatePath: dep.yaml
      templateFillMap:
        NumReplicas: 300              # 대규모 백엔드

  # Medium 백엔드 ILB (150 Pods)
  - replicasPerNamespace: {{$MEDIUM_BACKEND_LB_SERVICE_COUNT}}
    objectBundle:
    - basename: medium-backends-service
      templateFillMap:
        ILBSizeLabel: ilb-medium
    - basename: medium-backends-dep
      templateFillMap:
        NumReplicas: 150

  # Small 백엔드 ILB (10 Pods)
  - replicasPerNamespace: {{$SMALL_BACKEND_LB_SERVICE_COUNT}}
    objectBundle:
    - basename: small-backends-service
      templateFillMap:
        ILBSizeLabel: ilb-small
    - basename: small-backends-dep
      templateFillMap:
        NumReplicas: 10
```

### 4.6 백엔드 크기별로 왜 나누나?

```
┌─────────────────────────────────────────────────────────────────┐
│              백엔드 크기에 따른 성능 특성 분석                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Small (10 Pods)                                                │
│  ├── 빠른 프로비저닝 예상                                        │
│  ├── 일반적인 마이크로서비스 규모                                │
│  └── 기준선(Baseline) 성능                                       │
│                                                                 │
│  Medium (150 Pods)                                              │
│  ├── 중간 규모 서비스                                            │
│  ├── Health Check 시간 증가                                     │
│  └── 백엔드 등록 시간 증가                                       │
│                                                                 │
│  Large (300 Pods)                                               │
│  ├── 대규모 서비스 한계 테스트                                   │
│  ├── Cloud Provider API 부하                                    │
│  └── 최악의 경우 성능 파악                                       │
│                                                                 │
│  비교하면:                                                       │
│  "백엔드가 10개일 때 1분 걸리는데, 300개면 몇 분 걸리나?"         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. 테스트 시나리오 4: NetworkPolicy 테스트

**경로**: `testing/load/modules/network-policy/`

### 5.1 이 테스트는 무엇을 측정하나?

> **"NetworkPolicy를 적용하면 실제로 효과가 발생하기까지 얼마나 걸리는가?"**

NetworkPolicy는 Pod 간 통신을 제어하는 방화벽 규칙입니다. 이 규칙이 적용되는 시간이 길면 보안 취약점이 될 수 있습니다.

### 5.2 두 가지 테스트 유형

#### Policy Creation 테스트

```
┌─────────────────────────────────────────────────────────────────┐
│                    Policy Creation 테스트                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  시나리오:                                                       │
│  "이미 실행 중인 Pod들에게 새로운 NetworkPolicy를 적용할 때"       │
│                                                                 │
│  ┌─────────┐         ┌─────────┐                               │
│  │ Client  │ ──X──►  │ Target  │   ← Deny Policy로 차단됨      │
│  │   Pod   │         │   Pod   │                               │
│  └─────────┘         └─────────┘                               │
│       │                   │                                     │
│       │   Allow Policy    │                                     │
│       │      생성         │                                     │
│       ▼                   │                                     │
│  ┌─────────┐         ┌─────────┐                               │
│  │ Client  │ ──────► │ Target  │   ← 이제 통신 가능!            │
│  │   Pod   │         │   Pod   │                               │
│  └─────────┘         └─────────┘                               │
│                                                                 │
│  측정: Allow Policy 생성 → 통신 성공까지 걸린 시간               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Pod Creation 테스트

```
┌─────────────────────────────────────────────────────────────────┐
│                      Pod Creation 테스트                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  시나리오:                                                       │
│  "NetworkPolicy가 이미 있는 상태에서 새 Pod이 생성될 때"           │
│                                                                 │
│  ┌─────────┐         ┌ ─ ─ ─ ─ ┐                               │
│  │ Client  │ Watch   │ Target  │   ← 아직 없음                  │
│  │   Pod   │ ───────►│   Pod   │                               │
│  └─────────┘         └ ─ ─ ─ ─ ┘                               │
│       │                   │                                     │
│       │  Target Pod 생성   │                                     │
│       ▼                   ▼                                     │
│  ┌─────────┐         ┌─────────┐                               │
│  │ Client  │ ──────► │ Target  │   ← 생성되자마자 통신 가능?     │
│  │   Pod   │         │   Pod   │                               │
│  └─────────┘         └─────────┘                               │
│                                                                 │
│  측정:                                                          │
│  1. Target Pod 생성 → IP 할당까지 시간                          │
│  2. IP 할당 → 통신 가능까지 시간 (Policy 적용 시간)              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 테스트 설정 분석

```yaml
# testing/load/modules/network-policy/net-policy-enforcement-latency.yaml

# Baseline 모드: 정책 없이 순수 지연시간 측정 (비교용)
{{$NETWORK_POLICY_ENFORCEMENT_LATENCY_BASELINE := false}}

# Target Pod 레이블 (이 레이블 가진 Pod에 대한 정책)
{{$NET_POLICY_ENFORCEMENT_LATENCY_TARGET_LABEL_VALUE := "enforcement-latency"}}

# 네임스페이스당 최대 Target Pod 수
{{$NET_POLICY_ENFORCEMENT_LATENCY_MAX_TARGET_PODS_PER_NS := 100}}

# 부하 정책 수 (정책 부하 시뮬레이션)
{{$NET_POLICY_ENFORCEMENT_LOAD_COUNT := 1000}}
# └── 동시에 1000개의 다른 정책도 생성하여 부하 상황 시뮬레이션

steps:
# Setup: 테스트 환경 구성
- name: Setup network policy enforcement latency measurement
  measurements:
  - Identifier: NetworkPolicyEnforcement
    Method: NetworkPolicyEnforcement
    Params:
      action: setup
      targetLabelValue: enforcement-latency
      baseline: false  # 정책 테스트 (true면 정책 없이)

# Run: 테스트 실행
- name: Run pod creation network policy enforcement latency measurement
  measurements:
  - Identifier: NetworkPolicyEnforcement
    Method: NetworkPolicyEnforcement
    Params:
      action: run
      testType: policy-creation  # 또는 pod-creation
      policyLoadCount: 1000      # 부하 정책 1000개
      policyLoadQPS: 10          # 초당 10개 정책 생성

# Complete: 정리
- name: Complete pod creation network policy enforcement latency measurement
  measurements:
  - Identifier: NetworkPolicyEnforcement
    Method: NetworkPolicyEnforcement
    Params:
      action: complete
      testType: policy-creation
```

### 5.4 Prometheus 메트릭 수집

```yaml
# testing/load/modules/network-policy/net-policy-metrics.yaml

# 정책 생성 지연시간 (Histogram)
- name: PolicyCreation - Perc99
  query: histogram_quantile(0.99, sum(policy_enforcement_latency_policy_creation_seconds_bucket) by (le))
  # └── "99%의 정책이 이 시간 내에 적용된다"

# Pod 생성 후 연결 가능 지연시간
- name: PodCreation - Perc99
  query: histogram_quantile(0.99, sum(rate(pod_creation_reachability_latency_seconds_bucket[%v])) by (le))

# Pod IP 할당 지연시간
- name: PodIpAssignedLatency - Perc99
  query: histogram_quantile(0.99, sum(rate(pod_ip_address_assigned_latency_seconds_bucket[%v])) by (le))
```

### 5.5 Cilium 관련 메트릭 (Cilium CNI 사용 시)

```yaml
# Cilium 정책 처리 성능
- name: Policy regeneration time - Perc99
  query: histogram_quantile(0.99, sum(cilium_policy_regeneration_time_stats_seconds_bucket{scope="total"}) by (le))
  # └── Cilium이 정책을 재생성하는 데 걸린 시간

- name: Endpoint regeneration latency - Perc99
  query: histogram_quantile(0.99, sum(cilium_endpoint_regeneration_time_stats_seconds_bucket{scope="total"}) by (le))
  # └── Endpoint(Pod)에 정책을 적용하는 데 걸린 시간

- name: Time between policy change and deployment - Perc99
  query: histogram_quantile(0.99, sum(cilium_policy_implementation_delay_bucket) by (le))
  # └── 정책 변경 → 실제 datapath 적용까지 시간

- name: Failed endpoint regenerations percentage
  query: sum(cilium_endpoint_regenerations_total{outcome="fail"}) / sum(cilium_endpoint_regenerations_total) * 100
  threshold: 0.01  # 0.01% 이상 실패하면 경고
```

---

## 6. 실행 방법과 결과 해석

### 6.1 네트워크 성능 테스트 실행

```bash
# TCP 1:1 테스트
./clusterloader2 \
  --kubeconfig=$HOME/.kube/config \
  --provider=gce \
  --testconfig=testing/network/config.yaml \
  --testoverrides=testing/network/tcp_protocol_override.yaml \
  --testoverrides=testing/network/1_1_ratio_override.yaml \
  --report-dir=/tmp/results

# 전체 테스트 Suite 실행 (6가지 조합)
./clusterloader2 \
  --kubeconfig=$HOME/.kube/config \
  --provider=gce \
  --testsuite=testing/network/suite.yaml \
  --report-dir=/tmp/results
```

### 6.2 LoadBalancer 테스트 실행

```bash
# External LB 테스트
./clusterloader2 \
  --kubeconfig=$HOME/.kube/config \
  --provider=gce \
  --testconfig=testing/l4lb/config.yaml \
  --enable-exec-service=true \
  --report-dir=/tmp/results

# Internal LB 테스트
./clusterloader2 \
  --kubeconfig=$HOME/.kube/config \
  --provider=gce \
  --testconfig=testing/l4ilb/config.yaml \
  --enable-exec-service=true \
  --report-dir=/tmp/results
```

**주의**: `--enable-exec-service=true`는 Service 도달성 테스트에 필수입니다.

### 6.3 결과 파일 위치

```
/tmp/results/
├── NetworkPerformanceMetrics_1:1_TCP_P2P_summary.json
├── NetworkPerformanceMetrics_50:50_UDP_P2P_summary.json
├── ServiceCreationLatency_LBServiceCreationLatency_summary.json
├── LoadBalancerNodeSyncLatency_NodeSyncLatency_summary.json
└── ...
```

### 6.4 결과 해석 예시

#### 네트워크 성능 결과

```json
{
  "version": "v1",
  "dataItems": [
    {
      "labels": { "Metric": "Throughput" },
      "data": {
        "Perc05": 3500.0,
        "Perc50": 4800.0,
        "Perc95": 5200.0
      },
      "unit": "kbytes/sec"
    }
  ]
}
```

**해석**:
- 중앙값(Perc50) 4800 kbytes/sec ≈ 38.4 Mbps
- 최악의 5%(Perc05)도 3500 kbytes/sec ≈ 28 Mbps
- Perc05/Perc50 = 0.73 → 성능 편차가 적당함 (0.7 이상이면 양호)

#### LoadBalancer 생성 결과

```json
{
  "dataItems": [
    {
      "labels": { "Metric": "create_to_available_loadbalancer" },
      "data": {
        "Perc50": 45.2,
        "Perc99": 120.5
      },
      "unit": "s"
    }
  ]
}
```

**해석**:
- 절반의 LB가 45초 내에 준비됨
- 99%의 LB가 120초(2분) 내에 준비됨
- GCP 기준 정상 범위 (30초~120초)

### 6.5 자주 발생하는 문제

| 증상 | 가능한 원인 | 확인 방법 |
|------|-------------|-----------|
| TCP Throughput이 낮음 | CNI 오버레이 오버헤드 | 동일 노드 내 테스트와 비교 |
| UDP Packet Loss 높음 | 네트워크 혼잡, 버퍼 오버플로우 | `dmesg | grep drop` |
| LB 생성 시간이 김 | Cloud Provider 할당량 초과 | Cloud Console에서 할당량 확인 |
| NodeSync 시간이 김 | Cloud Provider API 지연 | Cloud Provider 로그 확인 |

---

## 부록: 파일 위치 요약

### 테스트 시나리오

| 테스트 | 설정 파일 | 템플릿 |
|--------|-----------|--------|
| 네트워크 성능 | `testing/network/config.yaml` | (내장) |
| L4 External LB | `testing/l4lb/config.yaml` | `service.yaml`, `dep.yaml` |
| L4 Internal LB | `testing/l4ilb/config.yaml` | `modules/`, `service.yaml`, `dep.yaml` |
| NetworkPolicy | `testing/load/modules/network-policy/*.yaml` | (내장) |

### 측정 구현체

| 측정 Method | 소스 파일 |
|-------------|-----------|
| NetworkPerformanceMetrics | `pkg/measurement/common/network/network_performance_measurement.go` |
| ServiceCreationLatency | `pkg/measurement/common/service_creation_latency.go` |
| LoadBalancerNodeSyncLatency | `pkg/measurement/common/loadbalancer_nodesync_latency.go` |
| NetworkPolicyEnforcement | `pkg/measurement/common/network-policy/network-policy-enforcement-latency.go` |

---

*이 문서는 ClusterLoader2 소스 코드를 기반으로 작성되었습니다.*
