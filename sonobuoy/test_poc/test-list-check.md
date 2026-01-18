## 모드별 실행 테스트 확인

### Quick 모드

      plugin-name: e2e
      result-format: junit
    spec:
      command:
--
      - name: E2E_FOCUS
        value: Pods should be submitted and removed
      - name: E2E_PARALLEL
        value: "false"
--
      plugin-name: systemd-logs
      result-format: raw
    spec:
      command:

### Non-Disruptive Conformance 모드

      plugin-name: e2e
      result-format: junit
    spec:
      command:
--
      - name: E2E_FOCUS
        value: \[Conformance\]
      - name: E2E_PARALLEL
        value: "false"
      - name: E2E_SKIP
        value: \[Disruptive\]|NoExecuteTaintManager
      - name: E2E_USE_GO_RUNNER
        value: "true"
--
      plugin-name: systemd-logs
      result-format: raw
    spec:
      command:
