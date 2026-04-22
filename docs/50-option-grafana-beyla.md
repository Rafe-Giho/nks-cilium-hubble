# Option 2. Grafana Beyla

- 목적: Cilium 외 sidecar-less 앱/트래픽 관측 후보로 Grafana Beyla 실행 절차와 한계를 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-04-22

## 이 옵션의 위치

Beyla는 CNI replacement가 아닙니다. Hubble 같은 네트워크 flow UI를 그대로 대체하지도 않습니다.

대신 eBPF 기반 auto-instrumentation으로 HTTP/gRPC RED metrics와 basic traces를 수집하는 데 적합합니다.

## 선택 조건

Beyla가 맞는 경우:

- 애플리케이션 RED metrics가 필요함
- 코드 수정 없이 basic traces를 보고 싶음
- Grafana, Prometheus, OTLP Collector와 연결할 계획이 있음

Beyla가 맞지 않는 경우:

- CNI/Ingress/Gateway 전환 검증이 목표
- Hubble 같은 네트워크 flow 중심 UX가 필요
- eBPF traffic control 충돌 리스크를 검토할 수 없음

## 0. 요구사항 확인

```bash
kubectl version
kubectl get no -o wide
helm version
```

Cilium과 같이 쓸 경우 확인할 것:

- 커널 TCX 지원 여부
- Cilium/Beyla eBPF attachment 충돌 여부
- 필요 시 Cilium `bpf-filter-priority` 조정 가능 여부

## 1. Helm repo 추가

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

## 2. 기본 배포

```bash
helm install beyla grafana/beyla \
  --namespace beyla \
  --create-namespace
```

확인:

```bash
kubectl get pod -n beyla
kubectl logs -n beyla -l app.kubernetes.io/name=beyla --tail=100
```

## 3. 관측 범위 제한 예시

샘플 namespace만 계측하려면 values 파일을 만듭니다.

```bash
cat > manifests/50-beyla-values.yaml <<'EOF'
config:
  data:
    discovery:
      instrument:
        - k8s_namespace: demo-a
    routes:
      unmatched: heuristic
EOF
```

적용:

```bash
helm upgrade beyla grafana/beyla \
  --namespace beyla \
  --values manifests/50-beyla-values.yaml
```

## 4. OTLP export 예시

OTel Collector가 있을 때만 사용합니다.

```yaml
env:
  OTEL_EXPORTER_OTLP_ENDPOINT: "http://otel-collector.monitoring:4318"
```

필요하면 Secret로 인증 헤더를 전달합니다.

## 5. 수집 결과 확인

연동 방식에 따라 확인 위치가 달라집니다.

- Prometheus scrape
- Grafana dashboard
- OTLP Collector
- trace backend

기본 확인:

```bash
kubectl get svc -n beyla
kubectl get pod -n beyla -o wide
kubectl logs -n beyla -l app.kubernetes.io/name=beyla --tail=100
```

## 성공 기준

- Beyla DaemonSet 정상
- demo namespace 트래픽 수집
- Prometheus 또는 OTLP 경로로 metrics/traces 확인
- Cilium과 함께 사용할 경우 eBPF 충돌 없음

## 한계

- Hubble UI 대체재가 아닙니다.
- Cilium CNI/Ingress/Gateway 기능을 제공하지 않습니다.
- 네트워크 flow보다는 앱 RED metrics/basic traces에 가깝습니다.

## 참고 출처

- https://grafana.com/docs/beyla/latest/
- https://grafana.com/docs/beyla/latest/setup/kubernetes-helm/
- https://grafana.com/docs/beyla/latest/cilium-compatibility/
- https://grafana.com/docs/beyla/latest/performance/
