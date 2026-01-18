# systemd-logs 플러그인 상세

## 실행 위치 및 구조

### DaemonSet 형태

| 항목 | 내용 |
|------|------|
| **타입** | DaemonSet |
| **목적** | 각 노드에서 systemd 로그 수집 |
| **실행 위치** | `docker-desktop` 노드 |
| **Pod 이름** | `sonobuoy-systemd-logs-daemon-set-7d99fc191af04ddb-9kks7` |

### 컨테이너 구성

Pod 내부에 **2개의 컨테이너**가 실행됩니다:

#### 1. systemd-logs 컨테이너

| 항목 | 내용 |
|------|------|
| **이미지** | `sonobuoy/systemd-logs:v0.4` |
| **역할** | 실제 로그 수집 수행 |
| **명령어** | `/get_systemd_logs.sh` 실행 |
| **마운트** | `/node` (호스트 루트 파일시스템) |

**실행 명령:**
```bash
/bin/sh -c /get_systemd_logs.sh; 
while true; do 
  echo "Plugin is complete. Sleeping indefinitely..."; 
  sleep 3600; 
done
```

#### 2. sonobuoy-worker 컨테이너

| 항목 | 내용 |
|------|------|
| **이미지** | `sonobuoy/sonobuoy:v0.57.3` |
| **역할** | 수집된 로그를 Aggregator에 전송 |
| **명령어** | `/sonobuoy worker single-node` |

**Worker 기능:**
- Aggregator와 통신하여 결과 전송
- 진행 상황 보고
- 결과 파일 관리

## 수집하는 로그

### 목적

**systemd-logs 플러그인**은 각 노드의 systemd/journalctl 로그를 수집합니다.

| 로그 소스 | 설명 |
|----------|------|
| **systemd journal** | systemd 서비스 로그 |
| **kubelet 로그** | Kubernetes kubelet 로그 |
| **컨테이너 런타임 로그** | Docker/containerd 로그 |
| **시스템 로그** | 노드 레벨 시스템 로그 |

### Docker Desktop 환경 제약

**현재 로그 확인:**
```
chroot: failed to run command 'journalctl': No such file or directory
Plugin is complete. Sleeping indefinitely...
```

**의미:**
- Docker Desktop Kubernetes는 systemd/journalctl을 사용하지 않음
- `journalctl` 명령이 없어서 로그 수집 실패
- 하지만 스크립트는 완료로 보고하고 계속 실행 중

**비고:**
- 실제 production Kubernetes (Linux 노드)에서는 정상 작동
- Docker Desktop 환경에서는 로그 수집 불가 (제한사항)

## 실행 흐름

```
1. DaemonSet 생성
   ↓
2. 각 노드에 Pod 생성 (docker-desktop)
   ↓
3. systemd-logs 컨테이너: /get_systemd_logs.sh 실행
   ├─ 노드의 /node 경로 마운트로 호스트 접근
   ├─ journalctl 명령으로 로그 수집 시도
   └─ 로그를 /tmp/sonobuoy/results에 저장
   ↓
4. sonobuoy-worker 컨테이너: 결과 전송
   ├─ 수집된 로그 파일 확인
   ├─ Aggregator에 결과 전송
   └─ 진행 상황 보고
   ↓
5. 완료 후 무한 루프로 대기
   └─ (완료 신호를 기다림)
```

## 현재 상태

### Pod 상태

| 컨테이너 | 상태 | Ready |
|---------|------|-------|
| systemd-logs | Running | ✅ True |
| sonobuoy-worker | Running | ✅ True |

### 로그 상태

**systemd-logs 컨테이너:**
- ✅ 스크립트 실행 완료 ("Plugin is complete")
- ⚠️ journalctl 명령 실패 (Docker Desktop 환경 제약)
- ⏳ 무한 루프로 대기 중

**sonobuoy-worker 컨테이너:**
- ✅ Aggregator와 통신 중
- ⏳ 결과 전송 대기

## 결과 위치

수집된 로그는 다음 위치에 저장됩니다:

```
/tmp/sonobuoy/results/
└── systemd-logs/
    └── <노드명>/
        └── <로그 파일들>
```

**최종 결과:**
- `sonobuoy retrieve` 명령으로 다운로드 가능
- `results.tar.gz` 파일 내부에 포함됨

## 관련 문서

- Sonobuoy 플러그인 문서: https://sonobuoy.io/docs/main/plugins/
- systemd-logs 플러그인: DaemonSet 타입 플러그인 예시
