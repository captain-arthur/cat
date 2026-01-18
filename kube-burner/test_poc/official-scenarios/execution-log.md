# 공식 시나리오 POC

실행 일시: 2026-01-18 18:20:33


### 1. 공식 시나리오 다운로드

시나리오: cluster-density
URL: https://raw.githubusercontent.com/cloud-bulldozer/kube-burner/main/examples/cluster-density/cluster-density.yml
다운로드 시간: 2026-01-18 18:20:37

### 2. 시나리오 내용 확인

jobIterations: 확인 필요
리소스 타입: 

### 3. 공식 시나리오 실행 (cluster-density)

실행 명령: kube-burner init -c cluster-density-official.yml
시작 시간: 2026-01-18 18:21:04
time="2026-01-18 18:21:04" level=error msg="Config error" file="kube-burner.go:129"
failed to parse config file: configuration file: 
line 28: field measurements not found in type config.Spec
### 3. 공식 시나리오 실행 (cluster-density - 간소화 버전)

실행 명령: kube-burner init -c cluster-density-official.yml
시작 시간: 2026-01-18 18:21:10
time="2026-01-18 18:21:10" level=info msg="🔥 Starting kube-burner (2.2.0@f4fa266a96875c8eb83f04f6fb265413deb1ab64) with UUID 5d7fa95d-e3f4-4237-9386-d9f511127730" file="job.go:85"
time="2026-01-18 18:21:10" level=warning msg="cluster-density: Invalid QPS (0); using default: 5" file="job.go:358"
time="2026-01-18 18:21:10" level=warning msg="cluster-density: Invalid Burst (0); using default: 10" file="job.go:365"
time="2026-01-18 18:21:10" level=fatal msg="Error reading template apiVersion: v1\nkind: Pod\nmetadata:\n  generateName: pod-\n  labels:\n    app: kube-burner-test\nspec:\n  containers:\n  - name: pause\n    image: registry.k8s.io/pause:3.9\n    resources:\n      requests:\n        cpu: 100m\n        memory: 128Mi\n: failed to open config file apiVersion: v1\nkind: Pod\nmetadata:\n  generateName: pod-\n  labels:\n    app: kube-burner-test\nspec:\n  containers:\n  - name: pause\n    image: registry.k8s.io/pause:3.9\n    resources:\n      requests:\n        cpu: 100m\n        memory: 128Mi\n: open apiVersion: v1\nkind: Pod\nmetadata:\n  generateName: pod-\n  labels:\n    app: kube-burner-test\nspec:\n  containers:\n  - name: pause\n    image: registry.k8s.io/pause:3.9\n    resources:\n      requests:\n        cpu: 100m\n        memory: 128Mi\n: no such file or directory" file="create.go:66"

### 4. 공식 시나리오 재실행 (템플릿 파일 분리)

시작 시간: 2026-01-18 18:22:05
time="2026-01-18 18:22:05" level=info msg="🔥 Starting kube-burner (2.2.0@f4fa266a96875c8eb83f04f6fb265413deb1ab64) with UUID ddb915e5-bd16-4c3a-a4db-bbd107c56427" file="job.go:85"
time="2026-01-18 18:22:05" level=warning msg="cluster-density: Invalid QPS (0); using default: 5" file="job.go:358"
time="2026-01-18 18:22:05" level=warning msg="cluster-density: Invalid Burst (0); using default: 10" file="job.go:365"
time="2026-01-18 18:22:05" level=info msg="Pre-load: images from job cluster-density" file="pre_load.go:73"
time="2026-01-18 18:22:05" level=info msg="Pre-load: Creating DaemonSet using images [registry.k8s.io/pause:3.9] in namespace preload-kube-burner" file="pre_load.go:203"
time="2026-01-18 18:22:05" level=info msg="Pre-load: Sleeping for 1m0s" file="pre_load.go:86"
time="2026-01-18 18:23:05" level=info msg="Deleting 1 namespaces with label: kube-burner-preload=true" file="namespaces.go:65"
time="2026-01-18 18:23:11" level=info msg="Initializing measurements for job: cluster-density" file="factory.go:104"
time="2026-01-18 18:23:11" level=info msg="Triggering job: cluster-density" file="job.go:115"
time="2026-01-18 18:23:11" level=info msg="Cleaning up previous runs" file="job.go:118"
time="2026-01-18 18:23:11" level=info msg="0/5 iterations completed" file="create.go:134"
time="2026-01-18 18:23:19" level=info msg="Waiting up to 4h0m0s for actions to be completed" file="create.go:260"
time="2026-01-18 18:23:20" level=info msg="Actions in namespace kube-burner-density-0 completed" file="waiters.go:77"
time="2026-01-18 18:23:21" level=info msg="Actions in namespace kube-burner-density-1 completed" file="waiters.go:77"
time="2026-01-18 18:23:22" level=info msg="Actions in namespace kube-burner-density-2 completed" file="waiters.go:77"
time="2026-01-18 18:23:25" level=info msg="Actions in namespace kube-burner-density-3 completed" file="waiters.go:77"

### 5. preload 이미지 문제 해결 (busybox로 변경)

문제: pause 이미지는 쉘/유틸리티가 없어서 preload 단계에서 실패
해결: pod-template에서 busybox:1.36 사용
시작 시간: 2026-01-18 18:24:14
time="2026-01-18 18:24:14" level=info msg="🔥 Starting kube-burner (2.2.0@f4fa266a96875c8eb83f04f6fb265413deb1ab64) with UUID 04b727bd-0cee-4ab1-b40c-3134f34053a4" file="job.go:85"
time="2026-01-18 18:24:14" level=warning msg="cluster-density: Invalid QPS (0); using default: 5" file="job.go:358"
time="2026-01-18 18:24:14" level=warning msg="cluster-density: Invalid Burst (0); using default: 10" file="job.go:365"
time="2026-01-18 18:24:14" level=info msg="Pre-load: images from job cluster-density" file="pre_load.go:73"
time="2026-01-18 18:24:14" level=info msg="Pre-load: Creating DaemonSet using images [busybox:1.36] in namespace preload-kube-burner" file="pre_load.go:203"
time="2026-01-18 18:24:14" level=info msg="Pre-load: Sleeping for 1m0s" file="pre_load.go:86"
time="2026-01-18 18:25:14" level=info msg="Deleting 1 namespaces with label: kube-burner-preload=true" file="namespaces.go:65"
time="2026-01-18 18:25:20" level=info msg="Initializing measurements for job: cluster-density" file="factory.go:104"
time="2026-01-18 18:25:20" level=info msg="Triggering job: cluster-density" file="job.go:115"
time="2026-01-18 18:25:20" level=info msg="Cleaning up previous runs" file="job.go:118"
time="2026-01-18 18:25:20" level=info msg="Deleting 5 namespaces with label: kube-burner.io/job=cluster-density" file="namespaces.go:65"
time="2026-01-18 18:25:36" level=info msg="0/5 iterations completed" file="create.go:134"
time="2026-01-18 18:25:44" level=info msg="Waiting up to 4h0m0s for actions to be completed" file="create.go:260"
time="2026-01-18 18:25:46" level=info msg="Actions in namespace kube-burner-density-0 completed" file="waiters.go:77"
time="2026-01-18 18:25:48" level=info msg="Actions in namespace kube-burner-density-1 completed" file="waiters.go:77"
time="2026-01-18 18:25:48" level=info msg="Actions in namespace kube-burner-density-2 completed" file="waiters.go:77"
time="2026-01-18 18:25:49" level=info msg="Actions in namespace kube-burner-density-3 completed" file="waiters.go:77"
