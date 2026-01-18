# kube-burner POC 실행 로그

실행 일시: 2026-01-18 18:05:03

### 1. 설치 확인

Version: 2.2.0
Git Commit: f4fa266a96875c8eb83f04f6fb265413deb1ab64
Build Date: 2026-01-12T15:20:57Z
Go Version: go1.23.12
OS/Arch: darwin arm64

### 2. 클러스터 상태 확인

Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   25h   v1.32.2

### 3. 간단한 워크로드 실행

실행 명령: kube-burner init -c simple-test.yml
시작 시간: 2026-01-18 18:05:05
time="2026-01-18 18:05:05" level=error msg="Config error" file="kube-burner.go:129"
error rendering configuration template: rendering error: template: :11:26: executing "" at <.JobIteration>: map has no entry for key "JobIteration"
### 오류 발생
- 설정 파일 템플릿 변수 문법 오류
- .JobIteration 변수가 인식되지 않음
- kube-burner 템플릿 변수 문법 확인 필요

### 4. 수정된 설정 파일로 재실행

설정 파일 수정: 템플릿 변수 제거 (generateName 사용)
time="2026-01-18 18:06:34" level=info msg="🔥 Starting kube-burner (2.2.0@f4fa266a96875c8eb83f04f6fb265413deb1ab64) with UUID fab3bfaf-fba0-490d-9b3b-9ca947a731e1" file="job.go:85"
time="2026-01-18 18:06:34" level=warning msg="simple-pod-test: Invalid QPS (0); using default: 5" file="job.go:358"
time="2026-01-18 18:06:34" level=warning msg="simple-pod-test: Invalid Burst (0); using default: 10" file="job.go:365"
time="2026-01-18 18:06:34" level=fatal msg="Error reading template apiVersion: v1\nkind: Pod\nmetadata:\n  name: test-pod\n  generateName: test-pod-\n  labels:\n    app: kube-burner-test\nspec:\n  containers:\n  - name: pause\n    image: registry.k8s.io/pause:3.9\n    resources:\n      requests:\n        cpu: 100m\n        memory: 128Mi\n: failed to open config file apiVersion: v1\nkind: Pod\nmetadata:\n  name: test-pod\n  generateName: test-pod-\n  labels:\n    app: kube-burner-test\nspec:\n  containers:\n  - name: pause\n    image: registry.k8s.io/pause:3.9\n    resources:\n      requests:\n        cpu: 100m\n        memory: 128Mi\n: open apiVersion: v1\nkind: Pod\nmetadata:\n  name: test-pod\n  generateName: test-pod-\n  labels:\n    app: kube-burner-test\nspec:\n  containers:\n  - name: pause\n    image: registry.k8s.io/pause:3.9\n    resources:\n      requests:\n        cpu: 100m\n        memory: 128Mi\n: no such file or directory" file="create.go:66"

### 5. 파일 경로 방식으로 재실행

설정 파일 수정: objectTemplate을 별도 파일로 분리
time="2026-01-18 18:07:04" level=info msg="🔥 Starting kube-burner (2.2.0@f4fa266a96875c8eb83f04f6fb265413deb1ab64) with UUID 38aa0c0b-dfad-4592-b10e-afd51d5f1f43" file="job.go:85"
time="2026-01-18 18:07:04" level=warning msg="simple-pod-test: Invalid QPS (0); using default: 5" file="job.go:358"
time="2026-01-18 18:07:04" level=warning msg="simple-pod-test: Invalid Burst (0); using default: 10" file="job.go:365"
time="2026-01-18 18:07:04" level=info msg="Pre-load: images from job simple-pod-test" file="pre_load.go:73"
time="2026-01-18 18:07:04" level=info msg="Pre-load: Creating DaemonSet using images [registry.k8s.io/pause:3.9] in namespace preload-kube-burner" file="pre_load.go:203"
time="2026-01-18 18:07:04" level=info msg="Pre-load: Sleeping for 1m0s" file="pre_load.go:86"
time="2026-01-18 18:08:04" level=info msg="Deleting 1 namespaces with label: kube-burner-preload=true" file="namespaces.go:65"
time="2026-01-18 18:08:10" level=info msg="Initializing measurements for job: simple-pod-test" file="factory.go:104"
time="2026-01-18 18:08:10" level=info msg="Triggering job: simple-pod-test" file="job.go:115"
time="2026-01-18 18:08:10" level=info msg="Cleaning up previous runs" file="job.go:118"
time="2026-01-18 18:08:10" level=info msg="1/10 iterations completed" file="create.go:134"
time="2026-01-18 18:08:10" level=info msg="2/10 iterations completed" file="create.go:134"
time="2026-01-18 18:08:10" level=info msg="3/10 iterations completed" file="create.go:134"
time="2026-01-18 18:08:10" level=info msg="4/10 iterations completed" file="create.go:134"
time="2026-01-18 18:08:10" level=info msg="5/10 iterations completed" file="create.go:134"
time="2026-01-18 18:08:10" level=info msg="6/10 iterations completed" file="create.go:134"
time="2026-01-18 18:08:10" level=info msg="7/10 iterations completed" file="create.go:134"
time="2026-01-18 18:08:10" level=info msg="8/10 iterations completed" file="create.go:134"
time="2026-01-18 18:08:10" level=info msg="9/10 iterations completed" file="create.go:134"
time="2026-01-18 18:08:10" level=info msg="Waiting up to 4h0m0s for actions to be completed" file="create.go:260"
time="2026-01-18 18:08:11" level=info msg="Actions in namespace kube-burner-test-4 completed" file="waiters.go:77"
time="2026-01-18 18:08:12" level=info msg="Actions in namespace kube-burner-test-1 completed" file="waiters.go:77"
time="2026-01-18 18:08:12" level=info msg="Actions in namespace kube-burner-test-0 completed" file="waiters.go:77"
time="2026-01-18 18:08:12" level=info msg="Actions in namespace kube-burner-test-3 completed" file="waiters.go:77"
time="2026-01-18 18:08:12" level=info msg="Actions in namespace kube-burner-test-5 completed" file="waiters.go:77"
time="2026-01-18 18:08:12" level=info msg="Actions in namespace kube-burner-test-6 completed" file="waiters.go:77"
time="2026-01-18 18:08:13" level=info msg="Actions in namespace kube-burner-test-7 completed" file="waiters.go:77"
time="2026-01-18 18:08:13" level=info msg="Actions in namespace kube-burner-test-2 completed" file="waiters.go:77"
time="2026-01-18 18:08:13" level=info msg="Actions in namespace kube-burner-test-8 completed" file="waiters.go:77"
time="2026-01-18 18:08:13" level=info msg="Actions in namespace kube-burner-test-9 completed" file="waiters.go:77"
time="2026-01-18 18:08:13" level=info msg="Verifying created objects" file="utils.go:161"
time="2026-01-18 18:08:13" level=info msg="Job simple-pod-test took 3s" file="job.go:188"
time="2026-01-18 18:08:13" level=info msg="Finished execution with UUID: 38aa0c0b-dfad-4592-b10e-afd51d5f1f43" file="job.go:260"
time="2026-01-18 18:08:13" level=info msg="👋 Exiting kube-burner 38aa0c0b-dfad-4592-b10e-afd51d5f1f43" file="kube-burner.go:89"

### 6. 실행 완료

완료 시간: 2026-01-18 18:10:19
실행 시간: 약 3초 (Pre-load 1분 제외)
상태: 성공 (리소스 생성/정리 완료)
메모: Prometheus 없어서 메트릭 수집 없음
