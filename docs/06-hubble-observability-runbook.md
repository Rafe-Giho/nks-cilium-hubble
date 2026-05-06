# Hubble 운영형 관측 가이드

- 목적: Hubble을 네트워크 플로우 모니터링 용도로 보는 방법을 PoC와 운영 레벨로 나누어 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-05-06

## 결론

Hubble을 Grafana로 볼 수 있습니다. 다만 Grafana는 Hubble UI의 서비스 맵 화면을 그대로 대체하는 것이 아니라, Prometheus가 수집한 Hubble metrics를 시계열 대시보드로 보여주는 방식입니다.

현재 `docs/05-calico-chaining-hubble-runbook.md`로 구성한 `Calico 유지 + Cilium generic-veth chaining + Hubble` 상태와 이 문서는 정합성이 맞습니다. 운영형 Grafana 구성은 CNI를 바꾸지 않고, 현재 chaining values를 유지한 채 metrics와 `ServiceMonitor`, dashboard ConfigMap만 추가하는 방식입니다.

역할 구분:

| 방식 | 용도 | 실무 판단 |
| --- | --- | --- |
| Hubble UI | 현재 시점의 service map, namespace별 flow 확인 | 장애 분석/트러블슈팅에 적합 |
| Hubble CLI | 특정 flow, drop, DNS, HTTP 이벤트 즉시 확인 | 운영자 진단 명령으로 적합 |
| Grafana | flow/drop/DNS/HTTP 지표 추세, 알람, 대시보드 | 상시 모니터링에 적합 |
| Hubble exporter | flow event를 로그로 저장/전송 | 감사, 장기 검색, Loki/로그 플랫폼 연계에 적합 |

따라서 PoC는 `Hubble UI + CLI`, 운영형은 `Hubble metrics + Prometheus/Grafana + 필요 시 exporter`를 기본으로 봅니다.

## 전제 조건

- Cilium이 정상 설치되어 있어야 합니다.
- Hubble Relay/UI가 활성화되어 있어야 합니다.
- Hubble Relay를 위해 Cilium 노드 간 `TCP 4244`가 열려 있어야 합니다. 현재 NKS worker self `TCP 1-65535`가 유지되면 충족됩니다.
- `hubble` CLI가 설치되어 있어야 합니다.
- Grafana로 보려면 Prometheus 또는 Prometheus Operator 기반 수집기가 있어야 합니다.
- Cilium/Hubble metrics를 켜면 Prometheus scrape용 port가 열립니다. 기본값 기준 Cilium agent `9962`, Hubble metrics `9965`, Cilium operator `9963`을 수집 경로에서 허용해야 합니다.

확인:

```bash
cilium status
hubble version
kubectl -n kube-system get deploy,ds,svc | grep -E 'hubble|cilium'
```

## 1. PoC 기본: port-forward로 Hubble UI/CLI 보기

PoC에서는 이 방식이 기본입니다. 공인 IP 노출이 없고, 권한 있는 운영자만 로컬에서 열기 때문에 가장 안전합니다.

Relay port-forward:

```bash
cilium hubble port-forward
```

NKS + Cilium chaining 검증 중 `cilium hubble port-forward`가 `connection reset by peer`로 끊긴 사례가 있었습니다. 이 경우 아래처럼 Relay Service를 직접 port-forward합니다.

```bash
kubectl -n kube-system port-forward svc/hubble-relay 4245:80 --address 127.0.0.1
```

다른 터미널에서 CLI 확인:

```bash
hubble status --server localhost:4245
hubble observe --server localhost:4245 --namespace cilium-chain-check --last 40
hubble observe --server localhost:4245 --namespace cilium-chain-check --protocol dns --last 40
hubble observe --server localhost:4245 --verdict DROPPED --last 40
```

`rpc error: code = Unavailable`, `EOF`, `connection refused`는 대개 port-forward 세션 종료입니다. port-forward를 다시 실행한 뒤 재조회합니다.

Hubble CLI가 Relay보다 낮은 버전이면 아래 경고가 나올 수 있습니다. 짧은 PoC 조회는 가능하지만, 장기 검증 전에는 Relay와 같은 버전으로 맞춥니다.

```text
Hubble CLI version is lower than Hubble Relay
```

UI 열기:

```bash
cilium hubble ui
```

브라우저가 자동으로 열리지 않으면 아래 주소로 접속합니다.

```text
http://localhost:12000
```

확인할 것:

- namespace별 service map이 보이는지
- Ingress Envoy에서 backend Pod로 가는 flow가 보이는지
- DNS flow가 보이는지
- dropped flow가 없는지 또는 의도한 정책 차단인지

## 2. 팀 공유 PoC: Hubble UI를 내부 Ingress로 노출

Hubble UI를 여러 사람이 웹으로 봐야 하면 port-forward 대신 Ingress를 쓸 수 있습니다. 단, 공인 IP에 무방비로 열면 안 됩니다.

권장 조건:

- NKS LoadBalancer 보안그룹에서 회사/VPN IP만 허용
- TLS 적용
- 가능하면 인증 프록시 또는 사내 SSO 앞단 구성
- 외부 전체 공개 금지

Hubble UI service type은 Cilium Helm reference 기준 `ClusterIP` 또는 `NodePort`입니다. UI를 NKS LoadBalancer로 직접 여는 방식보다, `ClusterIP` service 뒤에 별도 Ingress Controller를 붙이는 구성이 더 적절합니다.

현재 05번 실행 가이드의 기본값은 Cilium Ingress Controller를 활성화하지 않습니다. 따라서 이 절은 `nginx-ingress`, NHN Cloud ALB Controller, 또는 별도 검증된 Ingress Controller가 있을 때만 진행합니다.

예시 values:

```yaml
hubble:
  relay:
    enabled: true
  ui:
    enabled: true
    service:
      type: ClusterIP
    ingress:
      enabled: true
      className: "<사용 중인 ingress class>"
      hosts:
        - hubble.example.internal
      annotations: {}
      tls: []
```

Cilium/Hubble values에서 추가 values를 생성합니다.

```bash
export HUBBLE_UI_HOST="hubble.example.internal"

cilium install \
  --version "${CILIUM_VERSION}" \
  --values manifests/05-cilium-chaining-values.yaml \
  --dry-run-helm-values \
  --set hubble.ui.ingress.enabled=true \
  --set hubble.ui.ingress.className="<사용 중인 ingress class>" \
  --set hubble.ui.ingress.hosts[0]="${HUBBLE_UI_HOST}" \
  --set hubble.ui.service.type=ClusterIP \
  > manifests/06-cilium-values-hubble-ui-ingress.yaml
```

적용:

```bash
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --values manifests/06-cilium-values-hubble-ui-ingress.yaml
```

확인:

```bash
cilium status --wait
kubectl -n kube-system get deploy,svc,ingress | grep -E 'hubble|cilium'
kubectl -n kube-system get ingress
```

접속은 DNS 또는 로컬 hosts에 `HUBBLE_UI_HOST`를 LoadBalancer 주소로 연결한 뒤 확인합니다.

```bash
curl -I -H "Host: ${HUBBLE_UI_HOST}" "http://<HUBBLE_UI_LB_ADDR>/"
```

중단 조건:

- LoadBalancer가 전체 인터넷에 열림
- TLS 또는 접근 제한 없이 운영자 외 사용자가 접근 가능
- Hubble UI 접근 로그/접근 제어 방안이 없음

## 3. 운영형: Grafana로 Hubble metrics 보기

Grafana는 Hubble의 실시간 flow event를 그대로 보여주는 것이 아니라, Prometheus가 scrape한 `hubble_` 계열 metrics를 대시보드와 알람으로 보여줍니다.

Cilium 공식 문서 기준 Cilium agent, Hubble, Cilium Operator metrics는 기본적으로 노출되지 않습니다. 활성화하면 Cilium agent `9962`, Hubble metrics `9965`, Cilium Operator `9963` 포트를 수집 대상으로 사용합니다.

대표적으로 볼 항목:

- 전체 flow 처리량
- dropped flow 추세
- DNS query/response 추세
- TCP/ICMP flow 추세
- HTTP request/response 추세
- 정책 차단 추세

주의: HTTP 같은 L7 metrics는 L7 protocol visibility가 적용된 트래픽에서만 의미 있게 나옵니다.

### 3-1. 운영형 PoC 구성 방식

이번 구성에서는 Cilium 예제 `monitoring-example.yaml`보다 `kube-prometheus-stack`을 우선 사용합니다. 이유는 운영 전환 시 `ServiceMonitor`, Grafana sidecar dashboard, Prometheus target 관리 방식이 더 현실적이기 때문입니다.

구성:

| 구성요소 | 역할 |
| --- | --- |
| `kube-prometheus-stack` | Prometheus Operator, Prometheus, Grafana 설치 |
| `manifests/06-kube-prometheus-stack-values.yaml` | Cilium/Hubble `ServiceMonitor`와 dashboard ConfigMap 수집 설정 |
| `manifests/06-cilium-hubble-metrics-values.yaml` | 05번 chaining 설정 유지 + Cilium/Hubble metrics + dashboard ConfigMap 활성화 |

전제:

- `docs/05-calico-chaining-hubble-runbook.md` 5번까지 통과
- `cilium status` 정상
- `hubble status --server localhost:4245` 또는 Hubble UI로 flow 확인 가능
- Helm 사용 가능
- 인터넷에서 Helm chart pull 가능

공통 변수:

```bash
export KUBECONFIG="/mnt/c/Users/user/Downloads/ta-sgh-vxlan-sidecarless-cls_kubeconfig.yaml"
export CILIUM_VERSION="1.19.3"
export KPS_VERSION="84.5.0"
```

### 3-2. kube-prometheus-stack 설치

먼저 `ServiceMonitor` CRD가 있는 Prometheus Operator 계열 관측 스택을 설치합니다.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update prometheus-community

kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version "${KPS_VERSION}" \
  --values manifests/06-kube-prometheus-stack-values.yaml
```

확인:

```bash
kubectl -n monitoring rollout status deploy/kube-prometheus-stack-operator --timeout=10m
kubectl -n monitoring rollout status deploy/kube-prometheus-stack-grafana --timeout=10m
kubectl -n monitoring get pod,svc
kubectl api-resources | grep servicemonitors
```

중단 조건:

- `ServiceMonitor` CRD가 생성되지 않음
- Prometheus Operator 또는 Grafana Pod가 Ready가 아님
- `monitoring` namespace의 주요 Pod가 `CrashLoopBackOff`

### 3-3. Cilium/Hubble metrics 활성화

현재 05번 chaining 설정을 유지하면서 metrics만 추가합니다.

```bash
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --values manifests/06-cilium-hubble-metrics-values.yaml
```

확인:

```bash
kubectl -n kube-system rollout status ds/cilium --timeout=10m
kubectl -n kube-system rollout status deploy/cilium-operator --timeout=10m
kubectl -n kube-system rollout status deploy/hubble-relay --timeout=10m
cilium status --wait
kubectl -n kube-system get svc hubble-metrics cilium-agent cilium-operator cilium-envoy -o wide
kubectl -n kube-system get servicemonitor
```

기대:

- `ds/cilium` Ready 2/2
- Hubble Relay/UI 유지
- `hubble-metrics` Service 생성
- `cilium-agent`, `cilium-operator`, `cilium-envoy`, `hubble` 계열 `ServiceMonitor` 생성
- 기존 Pod IP 대역은 계속 Calico `10.100.x.x`

중단 조건:

- 신규 Pod 생성 실패
- `cilium status` 비정상
- `v1beta1.metrics.k8s.io` 비정상화
- 기존 `cilium-chain-check/curl` Pod에서 CoreDNS/API 통신 실패

### 3-4. 트래픽 생성

Grafana/Prometheus에서 볼 데이터가 생기도록 기존 검증 Pod에서 트래픽을 발생시킵니다.

```bash
kubectl -n cilium-chain-check exec curl -- \
  curl -sS -m 5 http://kube-dns.kube-system.svc.cluster.local:9153/metrics >/dev/null

kubectl -n cilium-chain-check exec curl -- \
  curl -sS -k -m 5 https://kubernetes.default.svc/healthz || true
```

`healthz`가 `401 Unauthorized`를 반환하면 정상입니다. 서비스까지 도달했고 인증만 없는 상태입니다.

DNS flow를 더 만들고 싶으면:

```bash
kubectl -n cilium-chain-check exec curl -- \
  sh -c 'for i in $(seq 1 20); do nslookup kubernetes.default.svc.cluster.local >/dev/null; done'
```

### 3-5. Grafana 접속

```bash
kubectl -n monitoring get secret kube-prometheus-stack-grafana \
  -o jsonpath='{.data.admin-user}' | base64 -d
echo

kubectl -n monitoring get secret kube-prometheus-stack-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d
echo

kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80 --address 127.0.0.1
```

브라우저:

```text
http://localhost:3000
```

Grafana에서 확인할 것:

- datasource `Prometheus` 자동 생성
- dashboard 검색에서 `Cilium`, `Hubble` 관련 dashboard 확인
- Cilium/Hubble dashboard panels가 `No data`만 나오지 않는지 확인
- `Hubble DNS`, `Hubble Networking`, `Hubble General Processing` 계열 panels 확인

주의:

- dashboard는 ConfigMap label `grafana_dashboard=1`을 Grafana sidecar가 읽어서 가져옵니다.
- dashboard가 바로 안 보이면 Grafana Pod 로그와 dashboard ConfigMap label을 확인합니다.
- HTTP dashboard는 L7 HTTP 관측 트래픽이 있어야 의미 있게 채워집니다. 현재 기본 검증 트래픽만으로는 DNS/TCP/flow/drop 중심으로 봅니다.

### 3-6. Prometheus target과 query 확인

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090 --address 127.0.0.1
```

브라우저:

```text
http://localhost:9090/targets
```

PromQL 예시:

```text
{__name__=~"hubble_.*"}
hubble_flows_processed_total
hubble_drop_total
hubble_dns_queries_total
hubble_tcp_flags_total
```

확인할 target:

- `kube-system/cilium-agent`
- `kube-system/cilium-operator`
- `kube-system/cilium-envoy`
- `kube-system/hubble`

### 3-7. 롤백

```bash
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --values manifests/05-cilium-chaining-values.yaml

kubectl -n kube-system rollout status ds/cilium --timeout=10m
cilium status --wait
```

Grafana/Prometheus까지 제거하려면:

```bash
helm -n monitoring uninstall kube-prometheus-stack
kubectl delete namespace monitoring --ignore-not-found
```

주의: `monitoring` namespace를 삭제하면 Prometheus/Grafana 데이터도 삭제됩니다.

### 3-8. Cilium 공식 예제 Prometheus/Grafana

짧은 기능 확인만 필요하면 Cilium 공식 예제도 사용할 수 있습니다. 다만 운영형 전환 관점에서는 위의 `kube-prometheus-stack` 방식을 우선합니다.

```bash
export CILIUM_GIT_REF="v${CILIUM_VERSION}"

kubectl apply -f "https://raw.githubusercontent.com/cilium/cilium/${CILIUM_GIT_REF}/examples/kubernetes/addons/prometheus/monitoring-example.yaml"
```

접속:

```bash
kubectl -n cilium-monitoring port-forward service/grafana 3000:3000 --address 127.0.0.1
kubectl -n cilium-monitoring port-forward service/prometheus 9090:9090 --address 127.0.0.1
```

## 4. 운영형: Hubble exporter로 flow 로그 저장

Grafana metrics는 추세와 알람에 좋지만, 개별 flow event 장기 검색에는 맞지 않습니다. 개별 flow를 장기 보관하려면 Hubble exporter를 사용하고, 로그 수집기가 해당 파일 또는 stdout을 가져가게 구성합니다.

기본 exporter values:

이미 3-3에서 Grafana metrics를 활성화한 상태라면 `manifests/06-cilium-hubble-metrics-values.yaml`을 base로 사용합니다. 이 파일에는 05번 chaining 설정과 metrics, `ServiceMonitor`, dashboard 설정이 모두 포함되어 있습니다.

```bash
cilium install \
  --version "${CILIUM_VERSION}" \
  --values manifests/06-cilium-hubble-metrics-values.yaml \
  --dry-run-helm-values \
  --set hubble.export.static.enabled=true \
  --set hubble.export.static.filePath=/var/run/cilium/hubble/events.log \
  --set hubble.export.fileMaxSizeMb=100 \
  --set hubble.export.fileMaxBackups=10 \
  --set hubble.export.fileCompress=true \
  > manifests/06-cilium-values-hubble-exporter.yaml
```

적용:

```bash
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --values manifests/06-cilium-values-hubble-exporter.yaml
```

확인:

```bash
kubectl -n kube-system rollout status ds/cilium
kubectl -n kube-system exec ds/cilium -- tail -f /var/run/cilium/hubble/events.log
```

drop/error만 저장하고 싶으면 먼저 raw filter를 생성합니다.

```bash
hubble observe --verdict DROPPED --verdict ERROR --print-raw-filters
```

그 뒤 `hubble.export.static.allowList`에 반영합니다. 운영에서는 전체 flow를 무제한 저장하지 말고, drop/error/중요 namespace 위주로 필터링합니다.

로그 활용 방향:

- Loki + Grafana Explore
- Elasticsearch/OpenSearch
- SIEM 또는 사내 로그 플랫폼
- 장애 분석용 단기 보관

## 5. 운영 노출 기준

권장:

- Hubble UI는 기본적으로 port-forward 또는 내부 Ingress로만 접근합니다.
- Grafana는 기존 관측망/VPN/SSO 뒤에 둡니다.
- Hubble Relay를 외부에 열어야 하면 `hubble.relay.service.type=LoadBalancer`가 가능하지만, 보안그룹 제한과 TLS/mTLS를 전제로 합니다.
- 장기 보관은 Hubble UI가 아니라 metrics/log pipeline으로 처리합니다.

비권장:

- Hubble UI를 공인 LoadBalancer로 직접 공개
- Hubble Relay를 인증 없이 공인 IP로 공개
- 모든 flow를 장기 저장하면서 필터/보관주기 없이 운영
- Grafana만 보고 개별 flow 원인 분석까지 해결하려는 구성

## 6. 최소 검증 체크리스트

PoC:

- `cilium status`에서 Hubble Relay가 `OK`
- `hubble status` 성공
- `hubble observe --namespace cilium-chain-check`에서 flow 확인
- `cilium hubble ui`로 UI 접근 성공

Grafana:

- `hubble-metrics` service 생성
- Prometheus target에 `hubble-metrics`가 `UP`
- Grafana에서 Hubble dashboard 확인
- Prometheus에서 `{__name__=~"hubble_.*"}` 결과 확인

Exporter:

- `/var/run/cilium/hubble/events.log`에 flow 기록
- drop/error filter 적용 여부 확인
- 로그 수집기가 파일 또는 stdout을 수집
- 저장량과 보관주기 확인

## 참고 출처

- https://docs.cilium.io/en/stable/observability/hubble/setup/
- https://docs.cilium.io/en/stable/observability/hubble/hubble-ui/
- https://docs.cilium.io/en/stable/observability/hubble/hubble-cli/
- https://docs.cilium.io/en/stable/observability/grafana/
- https://docs.cilium.io/en/stable/observability/metrics/
- https://docs.cilium.io/en/stable/observability/hubble/configuration/export/
- https://docs.cilium.io/en/stable/helm-reference/
