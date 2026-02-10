# ClusterLoader2 Sonobuoy 플러그인용 이미지
# - perf-tests에서 clusterloader2 빌드
# - in-cluster 실행 (--run-from-cluster=true, --provider=skeleton)
# - 결과를 /tmp/results에 출력해 Sonobuoy가 수집

FROM golang:1.24-alpine AS builder
RUN apk add --no-cache git
WORKDIR /src
# Kubernetes perf-tests (clusterloader2 포함) 클론
ARG PERF_TESTS_REV=master
RUN git clone --depth 1 https://github.com/kubernetes/perf-tests.git . \
  && cd clusterloader2 && go build -o clusterloader2 ./cmd/

FROM alpine:3.19
RUN apk add --no-cache ca-certificates
COPY --from=builder /src/clusterloader2/clusterloader2 /usr/local/bin/
COPY config/config.yaml /config/config.yaml
COPY config/deployment.yaml /config/deployment.yaml

# Sonobuoy가 결과를 수집하는 경로
ENV REPORT_DIR=/tmp/results
VOLUME ["/tmp/results"]

# in-cluster 실행: kubeconfig 비우고 run-from-cluster=true, provider=skeleton
ENTRYPOINT ["/usr/local/bin/clusterloader2"]
CMD ["--run-from-cluster=true", "--provider=skeleton", "--testconfig=/config/config.yaml", "--report-dir=/tmp/results"]
