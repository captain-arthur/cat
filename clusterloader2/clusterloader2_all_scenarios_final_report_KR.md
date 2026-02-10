# Clusterloader2 전체 테스트 시나리오 최종 보고서 (GWT 기반)

## 1. 개요

본 문서는 `clusterloader2/testing` 디렉토리에 존재하는 모든 시나리오 파일(`.yaml`)의 목적과 역할을 GWT(Given-When-Then) 형식으로 정리한 최종 보고서입니다. 각 테스트 카테고리(폴더) 내의 개별 파일 단위로 설명을 제공하여, 전체 테스트 스위트에 대한 포괄적인 이해를 돕는 것을 목표로 합니다.

---

## 2. 시나리오 상세 분석

### `testing/access-tokens`
- **목표**: 서비스 어카운트 토큰 인증 관련 API 서버 성능 스트레스 테스트.
- **`config.yaml`**:
    - **Given**: 다수의 토큰과 서비스 어카운트, 그리고 토큰을 사용할 파드가 정의된 클러스터.
    - **When**: 파드가 마운트된 토큰을 사용하여 API 서버에 대량의 요청을 보냄.
    - **Then**: API 서버의 응답 시간이 SLO를 만족해야 함.
- **`deployment.yaml`, `serviceAccount.yaml`, `role.yaml`, `roleBinding.yaml`, `token.yaml`**:
    - `config.yaml` 시나리오 실행에 필요한 리소스 템플릿 파일들입니다. (Deployment, ServiceAccount, Role 등)

### `testing/batch`
- **목표**: 대량의 배치 작업(Job) 실행 시의 클러스터 성능 측정.
- **`config.yaml`**:
    - **Given**: `job.yaml` 템플릿과 함께 생성할 Job의 개수가 설정된 클러스터.
    - **When**: 설정된 수의 Job을 동시에 생성.
    - **Then**: Job 완료 시간(`JobCompletionLatency`)이 측정되고, API 서버 성능이 SLO를 만족해야 함.
- **`job.yaml`**:
    - `config.yaml`에서 사용할 Job의 템플릿.

### `testing/chaosmonkey`
- **목표**: 의도적인 장애 주입을 통해 클러스터의 회복탄력성 검증.
- **`override.yaml`**:
    - **Given**: 워크로드가 실행 중인 클러스터.
    - **When**: 이 파일을 오버라이드로 적용하여 카오스 테스트를 활성화하고, 노드/파드 장애를 주입.
    - **Then**: 장애 상황에서도 서비스가 안정적으로 유지되고 SLO를 만족해야 함.
- **`ignore_node_killer_container_restarts_100.yaml`**:
    - **Given**: `node-killer` 카오스 테스트가 실행 중인 클러스터.
    - **When**: 이 파일을 오버라이드로 적용.
    - **Then**: 노드 장애로 인한 특정 컨테이너의 재시작은 테스트 결과 계산에서 제외됨.

### `testing/density`
- **목표**: 클러스터의 최대 파드 수용량 및 고밀도 상태에서의 성능 검증.
- **`config.yaml`**:
    - **Given**: `deployment.yaml` 기반의 기본 밀도 테스트 설정.
    - **When**: 클러스터 크기에 맞춰 대량의 파드를 생성.
    - **Then**: API 서버 응답 시간과 파드 시작 시간이 SLO를 만족해야 함.
- **`high-density-config.yaml`**:
    - **Given**: 기본 설정보다 노드당 파드 수가 더 높게 설정된 고밀도 테스트.
    - **When**: 기본 설정보다 많은 파드를 생성.
    - **Then**: 더 높은 부하에서도 API 서버와 파드 시작 시간이 SLO를 만족해야 함.
- **`scheduler-suite.yaml`**:
    - **Given**: Affinity, Anti-Affinity 등 복잡한 스케줄링 조건이 포함된 파드 템플릿.
    - **When**: 복잡한 제약 조건의 파드를 대량으로 생성.
    - **Then**: 모든 파드가 성공적으로 스케줄링되고, 스케줄러 성능 및 클러스터 전반의 성능이 SLO를 만족해야 함.
- **`deployment.yaml`**:
    - 밀도 테스트에서 파드를 생성하기 위한 기본 Deployment 템플릿.
- **`scheduler/*/*.yaml`**:
    - `scheduler-suite.yaml`에서 사용하는 세부적인 스케줄링(Pod-Affinity, Anti-Affinity, Topology-Spread) 관련 파드 템플릿과 오버라이드 설정.

### `testing/dra` & `testing/dra-baseline`
- **목표**: 쿠버네티스의 동적 리소스 할당(DRA) 기능의 성능을 테스트하고, 일반 방식(baseline)과 비교.
- **`config.yaml`**:
    - **Given**: DRA 컨트롤러가 설치된 클러스터.
    - **When**: 동적 리소스를 요청하는 Job 또는 Long-running Job을 생성.
    - **Then**: 리소스 할당/해제에 걸리는 시간과 시스템 부하를 평가.
- **`job.yaml`, `long-running-job.yaml`, `resourceclaimtemplate.yaml`**:
    - DRA 테스트에 사용되는 Job, Long-running Job, 리소스 요청 템플릿. `dra-baseline` 폴더의 파일들도 동일한 역할을 수행하나, DRA를 사용하지 않는 대조군 역할을 함.

### `testing/experimental`
- **목표**: 실험적인 기능이나 아직 개발 중인 테스트 시나리오들을 포함.
- **`storage/**/*.yaml`**:
    - 스토리지와 관련된 다양한 실험적 테스트. 예를 들어, 여러 종류의 볼륨(CSI, emptydir, secret 등)을 사용하는 파드의 시작 시간을 측정하거나, 노드당 최대 볼륨 개수 시나리오를 테스트. 각 파일은 특정 볼륨 타입이나 시나리오에 대한 설정 또는 리소스 템플릿.

### `testing/experiments`
- **목표**: 특정 테스트 동작을 변경하거나 새로운 측정 방식을 시험하기 위한 설정 조각들.
- **`*.yaml`**:
    - 각 파일은 특정 기능을 켜거나(e.g., `enable_restart_count_check.yaml`), 특정 에러를 무시하거나(e.g., `ignore_known_gce_container_restarts.yaml`), 측정 방식을 변경(e.g., `use_simple_latency_query.yaml`)하는 오버라이드 설정.

### `testing/huge-service`
- **목표**: 수천 개의 엔드포인트를 가진 대규모 서비스의 성능 검증.
- **`config.yaml`**:
    - **Given**: `modules/statefulset.yaml`을 통해 생성된 수천 개의 파드와, 이 파드들을 엔드포인트로 가지는 `modules/service.yaml` 서비스.
    - **When**: 클라이언트가 이 대규모 서비스로 접속을 시도.
    - **Then**: 서비스 디스커버리 및 연결 수립 시간이 SLO를 만족하고, kube-proxy의 성능 저하가 없는지 확인.
- **`modules/*.yaml`**:
    - 대규모 서비스를 구성하기 위한 StatefulSet, Service, 측정 방법 템플릿.

### `testing/l4ilb` & `testing/l4lb`
- **목표**: L4 내부(ILB) 및 외부(LB) 로드밸런서의 성능, 트래픽 분배, 장애 복구 능력 테스트.
- **`config.yaml`**:
    - **Given**: 백엔드 파드들과 LoadBalancer 타입의 서비스.
    - **When**: 외부 클라이언트가 로드밸런서로 트래픽을 보내는 동안 백엔드 파드를 변경.
    - **Then**: 트래픽 처리량, 지연 시간, 장애 복구 시간이 SLO를 만족해야 함.
- **`config-ilb-recovery.yaml` (l4ilb)**:
    - **Given**: ILB 서비스와 백엔드 파드.
    - **When**: 백엔드 파드에 장애를 주입한 후 복구.
    - **Then**: ILB가 트래픽을 정상적으로 다시 전달하기까지의 복구 시간을 측정.
- **`dep.yaml`, `service.yaml`, `modules/*.yaml`**:
    - 로드밸런서 테스트에 필요한 Deployment, Service, 측정 방법 템플릿.

### `testing/list`
- **목표**: 대량의 오브젝트에 대한 `LIST` API 호출 성능 측정.
- **`config.yaml`**:
    - **Given**: `modules/list-benchmark.yaml`에 정의된 대로 대량의 오브젝트(e.g., ConfigMap)가 생성된 클러스터.
    - **When**: 해당 오브젝트들에 대한 `LIST` API 호출을 실행.
    - **Then**: `LIST` API 호출의 응답 시간이 SLO를 만족해야 함.
- **`*.yaml`**:
    - 테스트 실행에 필요한 역할(Role), 리소스, 측정 방법 등을 정의한 템플릿.

### `testing/load`
- **목표**: 다양한 유형의 워크로드를 시뮬레이션하여 클러스터 전반의 성능을 측정하는 종합 부하 테스트.
- **`config.yaml`**:
    - **Given**: `modules` 디렉토리의 다양한 워크로드 템플릿들이 조합된 테스트 설정.
    - **When**: CPU/메모리 부하, 서비스 생성, DNS 조회, NetworkPolicy 적용 등 다양한 부하를 동시에 발생시킴.
    - **Then**: 클러스터의 핵심 지표(API 응답성, 파드 시작 시간, DNS 응답 시간 등)가 전반적으로 안정적이며 SLO를 만족해야 함.
- **`modules/**/*.yaml`**:
    - `load` 테스트를 구성하는 개별 단위. DNS, NetworkPolicy, 서비스, ConfigMap/Secret 등 특정 영역에 부하를 주거나 성능을 측정하기 위한 모듈화된 템플릿들.

### `testing/neg`
- **목표**: GKE의 NEG(Network Endpoint Group) 로드밸런싱 성능 테스트.
- **`config.yaml`**:
    - **Given**: NEG와 연결된 GKE 인그레스(Ingress) 또는 로드밸런서.
    - **When**: 외부에서 대량의 트래픽을 전송.
    - **Then**: NEG를 통한 트래픽 분산이 정상 작동하고 응답 시간 및 처리량이 SLO를 만족해야 함.
- **`*.yaml`**:
    - NEG 테스트에 필요한 Deployment, Service, Ingress, 측정 방법 템플릿.

### `testing/node-throughput`
- **목표**: 단일 노드가 감당할 수 있는 최대 파드 수와 그 상태에서의 노드 성능 측정.
- **`config.yaml`**:
    - **Given**: 테스트를 진행할 단일 노드.
    - **When**: `rc.yaml` 템플릿을 사용하여 해당 노드에 파드를 집중적으로 생성.
    - **Then**: 노드가 다운되지 않고 안정성을 유지하며, 노드의 리소스 사용량이 한계 내에 있어야 함.
- **`rc.yaml`**:
    - 테스트에 사용될 파드를 정의하는 ReplicationController 템플릿.

### `testing/overrides`
- **목표**: 다른 테스트와 함께 사용되어 특정 파라미터를 덮어쓰는 설정 파일 모음.
- **`*.yaml`**:
    - 각 파일은 특정 환경(e.g., `kubemark_500_nodes.yaml` for 500-node Kubemark)이나 구성(e.g., `node_containerd.yaml` for containerd runtime)에 맞는 파라미터 값을 제공. 단독으로 실행되지 않음.

### `testing/prometheus`
- **목표**: Clusterloader2와 연동된 프로메테우스의 성능 및 설정 테스트.
- **`*.yaml`**:
    - 각 파일은 특정 타겟(etcd, anet, node-exporter 등)의 메트릭을 수집(scrape)하거나 수집하지 않도록 하는 설정을 정의.

### `testing/request-benchmark`
- **목표**: API 서버의 특정 엔드포인트에 대한 순수 처리 성능 벤치마크.
- **`config.yaml`**:
    - **Given**: `modules/benchmark-deployment.yaml`에 정의된 벤치마크 클라이언트 파드.
    - **When**: 클라이언트가 API 서버의 특정 엔드포인트에 직접 대량의 요청을 보냄.
    - **Then**: 해당 엔드포인트의 QPS, 지연 시간 등 순수 처리 성능이 측정됨.
- **`*.yaml`**:
    - 벤치마크 실행에 필요한 Deployment, ConfigMap, RBAC, 측정 방법 템플릿.

### `testing/watch-list`
- **목표**: 대량의 리소스에 대한 `WATCH` 성능 측정.
- **`config.yaml`**:
    - **Given**: `secret.yaml`을 통해 생성된 다수의 Secret 리소스와, 이를 감시(watch)하는 `job.yaml` 파드.
    - **When**: Secret 리소스에 변경을 가함.
    - **Then**: `WATCH` 이벤트가 파드에 도달하는 지연 시간이 SLO를 만족해야 함.
- **`*.yaml`**:
    - `WATCH` 테스트에 필요한 Job, Secret, RBAC 템플릿.

### `testing/windows-tests`
- **목표**: Windows 노드의 기능 및 성능 검증.
- **`config.yaml`**:
    - **Given**: Windows 노드가 포함된 클러스터와 `windows_override.yaml` 설정.
    - **When**: `rc.yaml` 템플릿을 사용하여 Windows 노드에 파드를 생성하고 부하를 줌.
    - **Then**: Windows 파드가 정상 실행되고, 기본적인 기능 및 성능이 기대치를 만족해야 함.
- **`rc.yaml`, `windows_override.yaml`**:
    - Windows 테스트에 사용될 파드 템플릿과 Windows 노드를 타겟팅하기 위한 오버라이드 설정.
