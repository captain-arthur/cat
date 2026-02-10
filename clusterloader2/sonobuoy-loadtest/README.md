# ClusterLoader2 + Sonobuoy 통합 부하 테스트

ClusterLoader2로 간단한 부하 테스트를 실행하고, **Sonobuoy**로 통합 실행·결과 수집·리포트를 수행하는 POC 폴더입니다.

## 목적

| 항목 | 설명 |
|------|------|
| **부하 테스트** | ClusterLoader2 (TestConfig 기반) |
| **실행 방식** | Sonobuoy 플러그인(Job)으로 클러스터 내 실행 |
| **결과** | Sonobuoy가 수집한 junit 결과로 `sonobuoy results`로 요약·리포트 |

## 폴더 구조

```
sonobuoy-loadtest/
├── README.md                 # 본 문서
├── Dockerfile                # ClusterLoader2 포함 플러그인 이미지 빌드
├── config/
│   └── simple-load.yaml      # 간단 부하 TestConfig (ns 2개, Pod 10개)
└── plugin/
    └── clusterloader2-sonobuoy-plugin.yaml   # Sonobuoy 플러그인 정의
```

## 사전 요구사항

- Kubernetes 클러스터 (kubectl 설정됨)
- Docker (이미지 빌드용)
- Sonobuoy CLI (`brew install sonobuoy` 또는 [릴리스](https://github.com/vmware-tanzu/sonobuoy/releases)에서 설치)
- 클러스터에 Sonobuoy 네임스페이스·리소스 생성 권한

## 1. 플러그인 이미지 빌드

ClusterLoader2 바이너리와 TestConfig가 포함된 이미지를 빌드합니다.

```bash
cd clusterloader2/sonobuoy-loadtest
docker build -t clusterloader2-sonobuoy:latest .
```

**로컬 클러스터(Docker Desktop, kind 등)** 에서 사용할 때는 빌드한 이미지를 클러스터가 쓰는 레지스트리/로컬에 넣어야 합니다.

- **Docker Desktop Kubernetes**: 같은 호스트에서 빌드하면 해당 이미지를 그대로 사용 가능.
- **kind**: `kind load docker-image clusterloader2-sonobuoy:latest`

## 2. Sonobuoy로 부하 테스트 실행

ClusterLoader2 플러그인을 포함해 실행합니다. (`-m quick`로 빠른 e2e 1개 + ClusterLoader2 함께 실행)

```bash
# ClusterLoader2 플러그인 + quick e2e 함께 실행 (Sonobuoy 이미지 v0.57.3 사용)
sonobuoy run \
  -m quick \
  --sonobuoy-image sonobuoy/sonobuoy:v0.57.3 \
  --plugin plugin/clusterloader2-sonobuoy-plugin.yaml \
  --wait
```

- `--sonobuoy-image sonobuoy/sonobuoy:v0.57.3`: Sonobuoy Aggregator/Worker 이미지 (기본 v0.57.4보다 한 단계 낮춤).
- `--wait`: 완료될 때까지 대기.
- 필요 시 `--timeout 600` 등으로 타임아웃(초) 지정.

실행 중 상태 확인:

```bash
sonobuoy status
```

## 3. 결과 다운로드

실행이 끝나면 결과 아카이브를 가져옵니다.

```bash
results=$(sonobuoy retrieve)
echo "결과 파일: $results"
```

## 4. 결과 확인 (Sonobuoy로 요약·리포트)

Sonobuoy가 ClusterLoader2의 junit 결과를 파싱해 요약합니다.

```bash
# ClusterLoader2 플러그인 결과 요약 (이쁘게 출력)
sonobuoy results $results --plugin clusterloader2
```

**출력 예시:**

```
Plugin: clusterloader2
Status: passed
Total: 3
Passed: 3
Failed: 0
Skipped: 0
```

상세 테스트별 결과(JSON):

```bash
sonobuoy results $results --plugin clusterloader2 --mode detailed
```

아카이브 내 원본 파일 확인:

```bash
tar -xzf $results
ls plugins/clusterloader2/results/
# junit.xml, generatedConfig_*.yaml, cl2-metadata.json 등
cat plugins/clusterloader2/results/junit.xml
```

## 5. 정리

테스트 후 Sonobuoy 리소스 삭제:

```bash
sonobuoy delete --wait
```

ClusterLoader2가 만든 테스트 네임스페이스는 기본적으로 자동 정리됩니다 (`deleteAutomanagedNamespaces`).

## TestConfig 요약

`config/simple-load.yaml`:

| 항목 | 값 |
|------|-----|
| 네임스페이스 | 1~2 (2개) |
| 네임스페이스당 Pod 수 | 5 (총 10개) |
| 이미지 | `registry.k8s.io/pause:3.9` |
| Measurements | PodStartupLatency, APIAvailability |

부하를 늘리려면 `config/simple-load.yaml`에서 `namespaceRange.max`, `replicasPerNamespace` 등을 수정한 뒤 이미지를 다시 빌드하면 됩니다.

## 참고

- [ClusterLoader2 설치 및 사용법](../03-설치및사용법.md)
- [Sonobuoy 커스터마이징](../../sonobuoy/05-커스터마이징.md)
- [Sonobuoy 결과 확인](https://sonobuoy.io/docs/main/results/)
