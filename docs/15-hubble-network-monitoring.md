# Hubble Network Flow Monitoring Guide

- 목적: Hubble을 네트워크 플로우 모니터링 용도로 보는 방법을 PoC와 운영 레벨로 나누어 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-04-22

## 결론

Hubble을 Grafana로 볼 수 있습니다. 다만 Grafana는 Hubble UI의 서비스 맵 화면을 그대로 대체하는 것이 아니라, Prometheus가 수집한 Hubble metrics를 시계열 대시보드로 보여주는 방식입니다.

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

다른 터미널에서 CLI 확인:

```bash
hubble status
hubble observe --namespace demo-a
hubble observe --namespace demo-a --protocol dns
hubble observe --namespace demo-a --protocol http
hubble observe --verdict DROPPED
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

Hubble UI service type은 Cilium Helm reference 기준 `ClusterIP` 또는 `NodePort`입니다. UI를 NKS LoadBalancer로 직접 여는 방식보다, `ClusterIP` service 뒤에 Ingress를 붙이는 구성이 더 적절합니다.

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
      className: cilium
      hosts:
        - hubble.example.internal
      annotations:
        ingress.cilium.io/loadbalancer-mode: dedicated
      tls: []
```

Track A 최종 values에서 추가 values를 생성합니다.

```bash
export HUBBLE_UI_HOST="hubble.example.internal"

cilium install \
  --version "${CILIUM_VERSION}" \
  --values manifests/10-cilium-values-final-ingress.yaml \
  --dry-run-helm-values \
  --set hubble.ui.ingress.enabled=true \
  --set hubble.ui.ingress.className=cilium \
  --set hubble.ui.ingress.hosts[0]="${HUBBLE_UI_HOST}" \
  --set hubble.ui.service.type=ClusterIP \
  > manifests/15-cilium-values-hubble-ui-ingress.yaml
```

적용:

```bash
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --values manifests/15-cilium-values-hubble-ui-ingress.yaml
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

Cilium 공식 예제 기준 Hubble metrics endpoint는 기본 port `9965`를 사용하며, `hubble.metrics.enabled` 값이 비어 있지 않으면 `hubble-metrics` headless service에 Prometheus scrape annotation이 붙습니다.

대표적으로 볼 항목:

- 전체 flow 처리량
- dropped flow 추세
- DNS query/response 추세
- TCP/ICMP flow 추세
- HTTP request/response 추세
- 정책 차단 추세

주의: HTTP 같은 L7 metrics는 L7 protocol visibility가 적용된 트래픽에서만 의미 있게 나옵니다.

### 3-1. PoC용 Prometheus/Grafana 예제 배포

기존 관측 스택이 없으면 Cilium 예제 Prometheus/Grafana를 사용할 수 있습니다.

```bash
export CILIUM_GIT_REF="v${CILIUM_VERSION}"

kubectl apply -f "https://raw.githubusercontent.com/cilium/cilium/${CILIUM_GIT_REF}/examples/kubernetes/addons/prometheus/monitoring-example.yaml"
```

Grafana 접속:

```bash
kubectl -n cilium-monitoring port-forward service/grafana 3000:3000
```

브라우저:

```text
http://localhost:3000
```

Prometheus 접속:

```bash
kubectl -n cilium-monitoring port-forward service/prometheus 9090:9090
```

브라우저:

```text
http://localhost:9090
```

### 3-2. Cilium/Hubble metrics 활성화

Track A 최종 values에 metrics 옵션을 더해 별도 values를 생성합니다.

```bash
cilium install \
  --version "${CILIUM_VERSION}" \
  --values manifests/10-cilium-values-final-ingress.yaml \
  --dry-run-helm-values \
  --set prometheus.enabled=true \
  --set operator.prometheus.enabled=true \
  --set hubble.enabled=true \
  --set hubble.metrics.enableOpenMetrics=true \
  --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}" \
  > manifests/15-cilium-values-hubble-metrics.yaml
```

적용:

```bash
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --values manifests/15-cilium-values-hubble-metrics.yaml
```

확인:

```bash
cilium status --wait
kubectl -n kube-system get svc hubble-metrics cilium-agent -o wide
kubectl -n kube-system get svc hubble-metrics -o yaml | grep -E 'prometheus.io/scrape|prometheus.io/port'
```

Prometheus에서 확인할 query 예시:

```text
{__name__=~"hubble_.*"}
hubble_flows_processed_total
hubble_drop_total
hubble_dns_queries_total
hubble_http_requests_total
```

Grafana에서는 Cilium 예제 dashboard 중 Hubble 관련 dashboard를 먼저 보고, 운영 기준에 맞게 drop rate, DNS error, HTTP 5xx, namespace별 traffic panels를 별도로 정리합니다.

### 3-3. 기존 Prometheus Operator를 쓰는 경우

기존 Prometheus Operator가 있으면 annotation scrape보다 `ServiceMonitor`를 쓰는 편이 운영에 적합합니다.

```bash
cilium install \
  --version "${CILIUM_VERSION}" \
  --values manifests/10-cilium-values-final-ingress.yaml \
  --dry-run-helm-values \
  --set prometheus.enabled=true \
  --set prometheus.serviceMonitor.enabled=true \
  --set operator.prometheus.enabled=true \
  --set operator.prometheus.serviceMonitor.enabled=true \
  --set hubble.enabled=true \
  --set hubble.metrics.enableOpenMetrics=true \
  --set hubble.metrics.serviceMonitor.enabled=true \
  --set hubble.metrics.enabled="{dns,drop,tcp,flow,icmp,httpV2}" \
  > manifests/15-cilium-values-hubble-servicemonitor.yaml
```

적용 전 확인:

```bash
kubectl api-resources | grep servicemonitors
```

`ServiceMonitor` CRD가 없으면 이 방식은 쓰지 않습니다.

## 4. 운영형: Hubble exporter로 flow 로그 저장

Grafana metrics는 추세와 알람에 좋지만, 개별 flow event 장기 검색에는 맞지 않습니다. 개별 flow를 장기 보관하려면 Hubble exporter를 사용하고, 로그 수집기가 해당 파일 또는 stdout을 가져가게 구성합니다.

기본 exporter values:

```bash
cilium install \
  --version "${CILIUM_VERSION}" \
  --values manifests/10-cilium-values-final-ingress.yaml \
  --dry-run-helm-values \
  --set hubble.enabled=true \
  --set hubble.export.static.enabled=true \
  --set hubble.export.static.filePath=/var/run/cilium/hubble/events.log \
  --set hubble.export.fileMaxSizeMb=100 \
  --set hubble.export.fileMaxBackups=10 \
  --set hubble.export.fileCompress=true \
  > manifests/15-cilium-values-hubble-exporter.yaml
```

적용:

```bash
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --values manifests/15-cilium-values-hubble-exporter.yaml
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
- `hubble observe --namespace demo-a`에서 flow 확인
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
