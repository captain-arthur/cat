## POC 실행 시작

실행 일시: 2026-01-18 17:28:05

### 1. 이전 실행 정리 완료

### 2. 이미지 버전 확인
- Sonobuoy CLI: v0.57.4
- Sonobuoy Worker Image: v0.57.3

### 3. Quick 모드 테스트 실행


실행 명령:
```bash
sonobuoy run --mode quick --sonobuoy-image sonobuoy/sonobuoy:v0.57.3 --wait
```


time="2026-01-18T17:28:08+09:00" level=info msg="create request issued" name=sonobuoy namespace= resource=namespaces
time="2026-01-18T17:28:08+09:00" level=info msg="create request issued" name=sonobuoy-serviceaccount namespace=sonobuoy resource=serviceaccounts
time="2026-01-18T17:28:08+09:00" level=info msg="create request issued" name=sonobuoy-serviceaccount-sonobuoy namespace= resource=clusterrolebindings
time="2026-01-18T17:28:08+09:00" level=info msg="create request issued" name=sonobuoy-serviceaccount-sonobuoy namespace= resource=clusterroles
time="2026-01-18T17:28:08+09:00" level=info msg="create request issued" name=sonobuoy-config-cm namespace=sonobuoy resource=configmaps
time="2026-01-18T17:28:08+09:00" level=info msg="create request issued" name=sonobuoy-plugins-cm namespace=sonobuoy resource=configmaps
time="2026-01-18T17:28:08+09:00" level=info msg="create request issued" name=sonobuoy namespace=sonobuoy resource=pods
time="2026-01-18T17:28:08+09:00" level=info msg="create request issued" name=sonobuoy-aggregator namespace=sonobuoy resource=services
17:28:28          PLUGIN             NODE    STATUS   RESULT   PROGRESS
17:28:28             e2e           global   running                    
17:28:28    systemd-logs   docker-desktop   running                    
17:28:28 
17:28:28 Sonobuoy is still running. Runs can take 60 minutes or more depending on cluster and plugin configuration.
17:28:48             e2e           global   complete            Passed:  0, Failed:  0, Remaining:  1
...
...
...
...
...
...
...
...
...
...
...
...
...
...
...
...

### 4. 상태 확인 (2026-01-18 17:35:08)

         PLUGIN     STATUS   RESULT   COUNT                                PROGRESS
            e2e   complete                1   Passed:  0, Failed:  0, Remaining:  1
   systemd-logs    running                1                                        

Sonobuoy is still running. Runs can take 60 minutes or more depending on cluster and plugin configuration.



### 5. 실행 중단 (2026-01-18 17:37:49)

Docker Desktop 환경 제약으로 인해 systemd-logs 플러그인이 무한 대기 상태.
journalctl 명령이 없어 로그 수집 실패 후 완료 신호를 보내지 못함.

중단 결정.

