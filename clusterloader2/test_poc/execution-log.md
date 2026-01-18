# ClusterLoader2 POC 실행 로그

실행 일시: 2026-01-18 18:46:27


### 1. ClusterLoader2 설치 확인

I0118 18:46:28.385521   70344 dns_performance-k8s-hostnames.go:58] Registering measurement: DNS Performance for K8s Hostnames
I0118 18:46:28.385593   70344 network_performance_measurement.go:90] Registering Network Performance Measurement
Usage of ./clusterloader2:
      --add_dir_header                                      If true, adds the file directory to the header of the log messages
      --alsologtostderr                                     log to standard error as well as files (no effect when -logtostderr=true)

### 2. 간단한 테스트 실행

실행 명령: ./clusterloader2 --testconfig=simple-test.yaml --report-dir=./reports
시작 시간: 2026-01-18 18:46:35
I0118 18:46:35.545346   70456 dns_performance-k8s-hostnames.go:58] Registering measurement: DNS Performance for K8s Hostnames
I0118 18:46:35.545411   70456 network_performance_measurement.go:90] Registering Network Performance Measurement
F0118 18:46:35.545667   70456 clusterloader.go:279] Error init provider: unsupported provider name: 
I0118 18:46:35.545673   70456 clusterloader.go:272] Listening on 8000
I0118 18:46:43.388051   70496 dns_performance-k8s-hostnames.go:58] Registering measurement: DNS Performance for K8s Hostnames
I0118 18:46:43.388142   70496 network_performance_measurement.go:90] Registering Network Performance Measurement
I0118 18:46:43.388500   70496 clusterloader.go:272] Listening on 8000
I0118 18:46:43.403495   70496 clusterloader.go:168] ClusterConfig.Nodes set to 1
I0118 18:46:43.405594   70496 clusterloader.go:174] ClusterConfig.MasterName set to docker-desktop
E0118 18:46:43.407593   70496 clusterloader.go:185] Getting master external ip error: didn't find any ExternalIP master IPs
I0118 18:46:43.409837   70496 clusterloader.go:192] ClusterConfig.MasterInternalIP set to [192.168.65.3]
I0118 18:46:43.409846   70496 clusterloader.go:300] Using config: {ClusterConfig:{KubeConfigPath:/Users/hooni/.kube/config RunFromCluster:false Nodes:1 Provider:0x14000271020 EtcdCertificatePath:/etc/srv/kubernetes/pki/etcd-apiserver-server.crt EtcdKeyPath:/etc/srv/kubernetes/pki/etcd-apiserver-server.key EtcdInsecurePort:2382 MasterIPs:[] MasterInternalIPs:[192.168.65.3] MasterName:docker-desktop DeleteStaleNamespaces:false DeleteAutomanagedNamespaces:true APIServerPprofByClientEnabled:true KubeletPort:10250 K8SClientsNumber:1 SkipClusterVerification:false} ReportDir:./reports ExecServiceConfig:{Enable:true ImageRegistry:registry.k8s.io} ModifierConfig:{OverwriteTestConfig:[] SkipSteps:[]} PrometheusConfig:{TearDownServer:true EnableServer:false EnablePushgateway:false ScrapeEtcd:false ScrapeNodeExporter:false ScrapeWindowsNodeExporter:false ScrapeKubelets:false ScrapeMasterKubelets:false ScrapeKubeProxy:true KubeProxySelectorKey:component ScrapeKubeStateMetrics:false ScrapeMetricsServerMetrics:false ScrapeNodeLocalDNS:false ScrapeAnet:false ScrapeNetworkPolicies:false ScrapeMastersWithPublicIPs:false APIServerScrapePort:443 SnapshotProject: AdditionalMonitorsPath: StorageClassProvisioner:kubernetes.io/gce-pd StorageClassVolumeType:pd-ssd PVCStorageClass:ssd ReadyTimeout:15m0s PrometheusMemoryRequest:10Gi} OverridePaths:[]}
I0118 18:46:43.413316   70496 framework.go:74] Creating framework with 1 clients and "/Users/hooni/.kube/config" kubeconfig.
I0118 18:46:43.424576   70496 framework.go:276] Applying templates for "manifest/exec_deployment.yaml"
W0118 18:47:03.486285   70496 imagepreload.go:93] No images specified. Skipping image preloading
I0118 18:47:03.488046   70496 clusterloader.go:454] Test config successfully dumped to: reports/generatedConfig_.yaml
F0118 18:47:03.488065   70496 clusterloader.go:388] Test compilation failed: [steps: Invalid value: 0: cannot be empty]

### 3. 수정된 설정으로 재실행

시작 시간: 2026-01-18 18:48:54
I0118 18:48:54.480935   70862 dns_performance-k8s-hostnames.go:58] Registering measurement: DNS Performance for K8s Hostnames
I0118 18:48:54.481010   70862 network_performance_measurement.go:90] Registering Network Performance Measurement
I0118 18:48:54.481271   70862 clusterloader.go:272] Listening on 8000
I0118 18:48:54.491900   70862 clusterloader.go:168] ClusterConfig.Nodes set to 1
I0118 18:48:54.493624   70862 clusterloader.go:174] ClusterConfig.MasterName set to docker-desktop
E0118 18:48:54.495228   70862 clusterloader.go:185] Getting master external ip error: didn't find any ExternalIP master IPs
I0118 18:48:54.497971   70862 clusterloader.go:192] ClusterConfig.MasterInternalIP set to [192.168.65.3]
I0118 18:48:54.497979   70862 clusterloader.go:300] Using config: {ClusterConfig:{KubeConfigPath:/Users/hooni/.kube/config RunFromCluster:false Nodes:1 Provider:0x140006103f0 EtcdCertificatePath:/etc/srv/kubernetes/pki/etcd-apiserver-server.crt EtcdKeyPath:/etc/srv/kubernetes/pki/etcd-apiserver-server.key EtcdInsecurePort:2382 MasterIPs:[] MasterInternalIPs:[192.168.65.3] MasterName:docker-desktop DeleteStaleNamespaces:false DeleteAutomanagedNamespaces:true APIServerPprofByClientEnabled:true KubeletPort:10250 K8SClientsNumber:1 SkipClusterVerification:false} ReportDir:./reports ExecServiceConfig:{Enable:true ImageRegistry:registry.k8s.io} ModifierConfig:{OverwriteTestConfig:[] SkipSteps:[]} PrometheusConfig:{TearDownServer:true EnableServer:false EnablePushgateway:false ScrapeEtcd:false ScrapeNodeExporter:false ScrapeWindowsNodeExporter:false ScrapeKubelets:false ScrapeMasterKubelets:false ScrapeKubeProxy:true KubeProxySelectorKey:component ScrapeKubeStateMetrics:false ScrapeMetricsServerMetrics:false ScrapeNodeLocalDNS:false ScrapeAnet:false ScrapeNetworkPolicies:false ScrapeMastersWithPublicIPs:false APIServerScrapePort:443 SnapshotProject: AdditionalMonitorsPath: StorageClassProvisioner:kubernetes.io/gce-pd StorageClassVolumeType:pd-ssd PVCStorageClass:ssd ReadyTimeout:15m0s PrometheusMemoryRequest:10Gi} OverridePaths:[]}
I0118 18:48:54.502791   70862 framework.go:74] Creating framework with 1 clients and "/Users/hooni/.kube/config" kubeconfig.
I0118 18:48:54.515023   70862 framework.go:276] Applying templates for "manifest/exec_deployment.yaml"
W0118 18:49:04.575640   70862 imagepreload.go:93] No images specified. Skipping image preloading
F0118 18:49:04.577563   70862 clusterloader.go:388] Test compilation failed: [config reading error: decoding failed: error unmarshaling JSON: while decoding JSON: json: cannot unmarshal object into Go struct field Phase.steps.phases.tuningSet of type string]

### 4. tuningSet을 문자열로 수정하여 재실행

시작 시간: 2026-01-18 18:52:25
(eval):1: command not found: timeout

### 4. tuningSet을 문자열로 수정하여 재실행

I0118 18:55:39.358737   71892 dns_performance-k8s-hostnames.go:58] Registering measurement: DNS Performance for K8s Hostnames
I0118 18:55:39.359326   71892 network_performance_measurement.go:90] Registering Network Performance Measurement
I0118 18:55:39.360196   71892 clusterloader.go:272] Listening on 8000
I0118 18:55:39.379912   71892 clusterloader.go:168] ClusterConfig.Nodes set to 1
I0118 18:55:39.382442   71892 clusterloader.go:174] ClusterConfig.MasterName set to docker-desktop
E0118 18:55:39.384628   71892 clusterloader.go:185] Getting master external ip error: didn't find any ExternalIP master IPs
I0118 18:55:39.386275   71892 clusterloader.go:192] ClusterConfig.MasterInternalIP set to [192.168.65.3]
I0118 18:55:39.386283   71892 clusterloader.go:300] Using config: {ClusterConfig:{KubeConfigPath:/Users/hooni/.kube/config RunFromCluster:false Nodes:1 Provider:0x1400027c740 EtcdCertificatePath:/etc/srv/kubernetes/pki/etcd-apiserver-server.crt EtcdKeyPath:/etc/srv/kubernetes/pki/etcd-apiserver-server.key EtcdInsecurePort:2382 MasterIPs:[] MasterInternalIPs:[192.168.65.3] MasterName:docker-desktop DeleteStaleNamespaces:false DeleteAutomanagedNamespaces:true APIServerPprofByClientEnabled:true KubeletPort:10250 K8SClientsNumber:1 SkipClusterVerification:false} ReportDir:./reports ExecServiceConfig:{Enable:true ImageRegistry:registry.k8s.io} ModifierConfig:{OverwriteTestConfig:[] SkipSteps:[]} PrometheusConfig:{TearDownServer:true EnableServer:false EnablePushgateway:false ScrapeEtcd:false ScrapeNodeExporter:false ScrapeWindowsNodeExporter:false ScrapeKubelets:false ScrapeMasterKubelets:false ScrapeKubeProxy:true KubeProxySelectorKey:component ScrapeKubeStateMetrics:false ScrapeMetricsServerMetrics:false ScrapeNodeLocalDNS:false ScrapeAnet:false ScrapeNetworkPolicies:false ScrapeMastersWithPublicIPs:false APIServerScrapePort:443 SnapshotProject: AdditionalMonitorsPath: StorageClassProvisioner:kubernetes.io/gce-pd StorageClassVolumeType:pd-ssd PVCStorageClass:ssd ReadyTimeout:15m0s PrometheusMemoryRequest:10Gi} OverridePaths:[]}
I0118 18:55:39.389487   71892 framework.go:74] Creating framework with 1 clients and "/Users/hooni/.kube/config" kubeconfig.
I0118 18:55:39.401079   71892 framework.go:276] Applying templates for "manifest/exec_deployment.yaml"
W0118 18:55:49.462248   71892 imagepreload.go:93] No images specified. Skipping image preloading
I0118 18:55:49.466615   71892 clusterloader.go:454] Test config successfully dumped to: reports/generatedConfig_simple-pod-test.yaml
I0118 18:55:49.466635   71892 clusterloader.go:243] --------------------------------------------------------------------------------
I0118 18:55:49.466645   71892 clusterloader.go:244] Running simple-test-v3.yaml
I0118 18:55:49.466649   71892 clusterloader.go:245] --------------------------------------------------------------------------------
W0118 18:55:49.501821   71892 simple_test_executor.go:186] Got errors during step execution: [reading template () for identifier error: reading error: read .: is a directory]
E0118 18:55:59.512662   71892 clusterloader.go:253] --------------------------------------------------------------------------------
E0118 18:55:59.512679   71892 clusterloader.go:254] Test Finished
E0118 18:55:59.512683   71892 clusterloader.go:255]   Test: simple-test-v3.yaml
E0118 18:55:59.512686   71892 clusterloader.go:256]   Status: Fail
E0118 18:55:59.512689   71892 clusterloader.go:258]   Errors: [reading template () for identifier error: reading error: read .: is a directory]
E0118 18:55:59.512692   71892 clusterloader.go:260] --------------------------------------------------------------------------------

JUnit report was created: /Users/hooni/Documents/github/cat/clusterloader2/test_poc/reports/junit.xml
F0118 18:56:14.527007   71892 clusterloader.go:420] 2 tests have failed!

### 5. objectTemplatePath 사용으로 재실행

시작 시간: 2026-01-18 18:56:48
I0118 18:56:48.497892   72166 dns_performance-k8s-hostnames.go:58] Registering measurement: DNS Performance for K8s Hostnames
I0118 18:56:48.497960   72166 network_performance_measurement.go:90] Registering Network Performance Measurement
I0118 18:56:48.498214   72166 clusterloader.go:272] Listening on 8000
I0118 18:56:48.509921   72166 clusterloader.go:168] ClusterConfig.Nodes set to 1
I0118 18:56:48.512533   72166 clusterloader.go:174] ClusterConfig.MasterName set to docker-desktop
E0118 18:56:48.515312   72166 clusterloader.go:185] Getting master external ip error: didn't find any ExternalIP master IPs
I0118 18:56:48.517103   72166 clusterloader.go:192] ClusterConfig.MasterInternalIP set to [192.168.65.3]
I0118 18:56:48.517111   72166 clusterloader.go:300] Using config: {ClusterConfig:{KubeConfigPath:/Users/hooni/.kube/config RunFromCluster:false Nodes:1 Provider:0x140003203b0 EtcdCertificatePath:/etc/srv/kubernetes/pki/etcd-apiserver-server.crt EtcdKeyPath:/etc/srv/kubernetes/pki/etcd-apiserver-server.key EtcdInsecurePort:2382 MasterIPs:[] MasterInternalIPs:[192.168.65.3] MasterName:docker-desktop DeleteStaleNamespaces:false DeleteAutomanagedNamespaces:true APIServerPprofByClientEnabled:true KubeletPort:10250 K8SClientsNumber:1 SkipClusterVerification:false} ReportDir:./reports ExecServiceConfig:{Enable:true ImageRegistry:registry.k8s.io} ModifierConfig:{OverwriteTestConfig:[] SkipSteps:[]} PrometheusConfig:{TearDownServer:true EnableServer:false EnablePushgateway:false ScrapeEtcd:false ScrapeNodeExporter:false ScrapeWindowsNodeExporter:false ScrapeKubelets:false ScrapeMasterKubelets:false ScrapeKubeProxy:true KubeProxySelectorKey:component ScrapeKubeStateMetrics:false ScrapeMetricsServerMetrics:false ScrapeNodeLocalDNS:false ScrapeAnet:false ScrapeNetworkPolicies:false ScrapeMastersWithPublicIPs:false APIServerScrapePort:443 SnapshotProject: AdditionalMonitorsPath: StorageClassProvisioner:kubernetes.io/gce-pd StorageClassVolumeType:pd-ssd PVCStorageClass:ssd ReadyTimeout:15m0s PrometheusMemoryRequest:10Gi} OverridePaths:[]}
I0118 18:56:48.520747   72166 framework.go:74] Creating framework with 1 clients and "/Users/hooni/.kube/config" kubeconfig.
I0118 18:56:48.532614   72166 framework.go:276] Applying templates for "manifest/exec_deployment.yaml"
