# Sidecar-less Observability Architecture Concepts

- 목적: Track C에서 검증한 Calico chaining, Cilium, Hubble, Prometheus/Grafana, Hubble exporter의 개념과 연계 구조를 이론적으로 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-05-06

## 요약

현재 검증한 구성은 `Cilium full replacement`가 아닙니다.

현재 구조는 아래와 같습니다.

```text
NKS 기본 Calico VXLAN
  - Pod IPAM
  - Pod veth 생성
  - Pod-to-Pod / Pod-to-Service 기본 네트워킹
  - VXLAN overlay

Cilium generic-veth chaining
  - Calico가 만든 veth 기반 Pod 네트워크 뒤에 연결
  - Cilium endpoint 인식
  - Hubble 기반 flow 관측 제공
  - kube-proxy replacement 비활성

Hubble
  - Cilium agent가 수집한 flow 이벤트를 CLI/UI/metrics/exporter로 노출

Prometheus/Grafana
  - Hubble flow 이벤트를 집계한 metrics를 장기 시계열로 저장/시각화

Hubble exporter
  - 개별 flow 이벤트를 파일 또는 stdout으로 내보내 로그 수집 파이프라인과 연계
```

핵심 판단:

- Calico는 여전히 주 CNI입니다.
- Cilium은 네트워크 주도권을 완전히 가져오지 않고, chaining된 관측 계층으로 동작합니다.
- Hubble UI/CLI는 실시간 진단에 적합합니다.
- Grafana는 flow event 원문이 아니라 집계된 metrics 추세와 알람에 적합합니다.
- Hubble exporter는 개별 flow 로그의 장기 저장, 검색, 감사에 적합합니다.

## 전체 구성도

```text
                kubectl / hubble CLI / Browser
                         |
                         | port-forward or internal access
                         v
                 +----------------+
                 | Hubble Relay   |
                 | svc 4245 ->    |
                 | agent 4244     |
                 +----------------+
                         ^
                         |
        +----------------+----------------+
        |                                 |
+----------------+                +----------------+
| Cilium Agent   |                | Cilium Agent   |
| node-0         |                | node-1         |
| Hubble server  |                | Hubble server  |
| eBPF observe   |                | eBPF observe   |
+----------------+                +----------------+
        ^                                 ^
        | generic-veth chaining           | generic-veth chaining
        v                                 v
+----------------+                +----------------+
| Calico CNI     |                | Calico CNI     |
| Pod IPAM       |                | Pod IPAM       |
| veth / VXLAN   |                | veth / VXLAN   |
+----------------+                +----------------+
        |                                 |
        +----------- Pod Traffic ---------+

Metrics path:

Cilium/Hubble metrics ServiceMonitor
        |
        v
Prometheus
        |
        v
Grafana dashboard

Exporter path:

Cilium Agent
        |
        v
/var/run/cilium/hubble/events.log
        |
        v
Log collector -> Loki/OpenSearch/SIEM -> Grafana Explore or search UI
```

## 구성요소별 역할

| 구성요소 | 현재 역할 | 소유하는 기능 | 운영상 의미 |
| --- | --- | --- | --- |
| NKS | 관리형 Kubernetes | control plane, node group, 기본 addon | 관리형 제약을 우선 고려 |
| Calico | 주 CNI | Pod IPAM, Pod veth, overlay VXLAN, 기본 네트워킹 | 삭제 대상 아님 |
| Cilium CNI | chained CNI | Calico 뒤에서 endpoint 인식, eBPF 기반 관측 | 주 CNI 대체가 아니라 보조 관측 계층 |
| Cilium Agent | node agent | endpoint 관리, eBPF program, Hubble server, metrics/exporter | 각 노드의 관측 핵심 |
| Cilium Operator | cluster operator | Cilium 리소스/상태 관리 | agent 보조 컴포넌트 |
| Cilium Envoy | L7 proxy component | L7 관측/정책/Ingress 기능에 사용 가능 | 현재 기본 flow 검증에서는 핵심 경로 아님 |
| Hubble Relay | flow aggregation | 각 Cilium agent의 Hubble server에 연결 | UI/CLI가 cluster-wide flow를 보기 위한 중간 계층 |
| Hubble UI | visual UI | service map, namespace별 flow 시각화 | 장애 분석, 흐름 파악 |
| Hubble CLI | operator CLI | flow 조회, 필터, raw filter 생성 | 즉시 진단과 검증에 적합 |
| Hubble metrics | Prometheus metrics | flow/drop/DNS/TCP/HTTP 집계 지표 | 상시 모니터링, 알람, 추세 분석 |
| Hubble exporter | flow log export | 개별 flow event 파일/stdout 출력 | 장기 검색, 감사, 로그 분석 |
| Prometheus Operator | metrics 운영 계층 | `ServiceMonitor` 기반 scrape 관리 | 운영형 Prometheus 관리를 단순화 |
| kube-prometheus-stack | 관측 스택 패키지 | Prometheus, Grafana, Operator, CRD | PoC에서 운영형에 가까운 관측 스택 |
| Grafana | 시각화 | dashboard, PromQL 시각화, Explore | metrics와 로그 UI의 중심 |

## Calico VXLAN과 주 CNI 역할

현재 NKS 클러스터의 기본 네트워킹은 Calico가 담당합니다.

Calico의 현재 역할:

- Pod IP 할당
- Pod network namespace와 veth 생성
- Pod 간 통신 경로 구성
- 노드 간 overlay 통신
- 기존 kube-system 구성과 NKS 기본 동작 유지

현재 확인한 기준:

- Calico backend: `vxlan`
- Calico datastore: `etcdv3`
- Pod IP 대역: `10.100.x.x`
- Calico VXLAN port: `UDP 4789`
- CNI config path: `/etc/cni/net.d/calico`

이 구조에서 Calico를 제거하면 안 됩니다. Calico는 여전히 주 CNI이고, Cilium은 이 뒤에 붙은 chaining 구성입니다.

## CNI chaining과 generic-veth

CNI chaining은 하나의 Pod 네트워크 설정 과정에서 여러 CNI plugin을 순차 실행하는 방식입니다.

현재 구성의 plugin 흐름:

```text
Pod 생성
  |
  v
Calico CNI
  - Pod IP 할당
  - veth 생성
  - route / iptables / VXLAN 경로 설정
  |
  v
portmap / bandwidth
  - HostPort, bandwidth capability 유지
  |
  v
Cilium CNI
  - generic-veth chaining mode
  - Calico가 만든 veth 기반 endpoint를 Cilium에 연결
  - Cilium/Hubble 관측 가능 상태로 등록
```

`generic-veth`가 가능한 이유는 Calico가 Pod 연결에 veth 모델을 사용하기 때문입니다.

중요한 특성:

- Cilium이 Pod IPAM을 담당하지 않습니다.
- Cilium이 Calico VXLAN을 대체하지 않습니다.
- Cilium이 kube-proxy를 대체하지 않습니다.
- Cilium은 Calico가 만든 endpoint에 붙어 관측과 일부 Cilium 기능을 제공합니다.
- 기존 Pod에는 자동 적용되지 않고, chaining 적용 이후 새로 생성된 Pod부터 관측 대상이 됩니다.

현재 values의 핵심:

```yaml
cluster:
  name: ta-sgh-vxlan-sidecarless-cls
cni:
  chainingMode: generic-veth
  customConf: true
  configMap: cni-configuration
  confPath: /etc/cni/net.d/calico
  exclusive: false
routingMode: native
kubeProxyReplacement: "false"
enableIPv4Masquerade: false
enableIdentityMark: false
```

`cni.exclusive=false`는 Cilium이 기존 CNI 설정을 독점적으로 지우거나 대체하지 않게 하는 의미입니다.

## Cilium의 현재 역할과 한계

현재 Cilium은 주 CNI가 아니라 chained CNI입니다.

현재 기대하는 기능:

- Cilium endpoint 생성
- Hubble flow 관측
- Cilium/Hubble metrics 노출
- Hubble exporter를 통한 flow 로그 출력

현재 기대하지 않는 기능:

- Cilium Pod IPAM
- Cilium VXLAN/Geneve 기반 주 datapath
- kube-proxy replacement
- Cilium Ingress/Gateway API 기반 north-south 트래픽 처리
- Calico 완전 제거

chaining에서 주의할 점:

- Cilium advanced feature 일부는 제한될 수 있습니다.
- L7 정책, 투명 암호화 등은 별도 검증 없이 운영 전제로 두지 않습니다.
- 기존 Pod는 재시작 전까지 Cilium endpoint/Hubble 관측 대상이 아닐 수 있습니다.
- 장애 시 Calico와 Cilium 중 어디가 문제인지 분리 진단해야 합니다.

## Hubble의 개념

Hubble은 Cilium의 observability layer입니다.

역할:

- Cilium datapath에서 관측한 flow event 제공
- source/destination identity, namespace, pod, service, L4/L7 정보 제공
- forwarded/dropped/error verdict 확인
- DNS, TCP, HTTP 등 protocol별 흐름 확인

Hubble 구성:

```text
Cilium Agent
  - Hubble server
  - node-local flow source
  - TCP 4244

Hubble Relay
  - 여러 Cilium Agent의 Hubble server에 연결
  - cluster-wide flow stream 제공
  - Hubble CLI/UI 접속 지점

Hubble UI
  - service map
  - namespace/service/workload 기준 시각화

Hubble CLI
  - flow query
  - filter
  - raw filter 생성
```

현재 검증한 흐름:

- `cilium-chain-check/curl` -> CoreDNS `:53` DNS flow
- `cilium-chain-check/curl` -> CoreDNS `:9153` metrics HTTP/TCP flow
- `cilium-chain-check/curl` -> `kubernetes.default.svc` API service flow

`401 Unauthorized`는 네트워크 실패가 아니라 API server까지 도달한 뒤 인증이 없는 상태입니다.

## Hubble UI와 Hubble CLI

Hubble UI와 CLI는 flow event를 직접 보는 진단 도구입니다.

| 도구 | 강점 | 한계 |
| --- | --- | --- |
| Hubble UI | service map, 시각적 흐름 파악, namespace 중심 탐색 | 장기 저장/알람 용도 아님 |
| Hubble CLI | 세밀한 필터, 즉시 확인, drop/error 진단 | 사람이 실행해야 하는 운영 명령 성격 |

port-forward가 끊기면 `EOF`, `connection refused`, `connection reset by peer`가 발생할 수 있습니다. 이 경우 Hubble 자체 장애로 단정하지 않고 port-forward 세션을 먼저 다시 확인합니다.

## Hubble metrics와 Grafana

Hubble metrics는 개별 flow event를 그대로 저장하는 기능이 아닙니다.

Hubble metrics는 flow event를 Prometheus metrics 형태로 집계합니다.

예시:

- flow 처리량
- dropped flow 수
- DNS query/response
- TCP flag
- ICMP
- HTTP request/response

흐름:

```text
Cilium Agent / Hubble metrics endpoint
  |
  | ServiceMonitor
  v
Prometheus
  |
  | PromQL
  v
Grafana dashboard
```

현재 values의 핵심:

```yaml
prometheus:
  enabled: true
  serviceMonitor:
    enabled: true

hubble:
  metrics:
    enableOpenMetrics: true
    enabled:
      - dns:query;ignoreAAAA
      - drop
      - tcp
      - flow
      - port-distribution
      - icmp
      - httpV2:...
    serviceMonitor:
      enabled: true
```

운영 특성:

- Grafana는 추세, 이상 징후, 알람에 적합합니다.
- Prometheus 보관 기간과 scrape interval에 따라 저장량이 결정됩니다.
- label cardinality가 높으면 Prometheus 부하가 커질 수 있습니다.
- HTTP metrics는 L7 visibility가 있는 트래픽에서 의미 있게 나옵니다.

## Prometheus Operator와 ServiceMonitor

Prometheus Operator는 Prometheus scrape 설정을 Kubernetes CRD로 관리하는 방식입니다.

`ServiceMonitor`의 역할:

- 어떤 Service를 scrape할지 선언
- 어떤 namespace를 볼지 선언
- 어떤 port/path/interval로 scrape할지 선언
- Prometheus 설정 파일을 직접 수정하지 않고 Kubernetes 리소스로 관리

현재 `kube-prometheus-stack`을 쓰는 이유:

- Prometheus Operator CRD 제공
- Prometheus instance 제공
- Grafana 제공
- 기본 Kubernetes dashboard 제공
- Grafana sidecar가 dashboard ConfigMap 자동 수집

현재 dashboard 연계:

```text
Cilium Helm chart
  |
  v
Grafana dashboard ConfigMap
  label: grafana_dashboard=1
  |
  v
Grafana sidecar
  |
  v
Grafana dashboard 자동 등록
```

## Hubble exporter

Hubble exporter는 개별 flow event를 파일 또는 stdout으로 내보내는 기능입니다.

흐름:

```text
Cilium Agent
  |
  v
/var/run/cilium/hubble/events.log
  |
  v
로그 수집기
  |
  v
Loki / OpenSearch / Elasticsearch / SIEM
  |
  v
Grafana Explore 또는 검색 UI
```

Grafana metrics와 exporter의 차이:

| 항목 | Hubble metrics | Hubble exporter |
| --- | --- | --- |
| 데이터 형태 | 집계 시계열 | 개별 flow event |
| 저장소 | Prometheus | 로그 플랫폼 |
| 주요 용도 | 대시보드, 알람, 추세 | 검색, 감사, 사건 분석 |
| 조회 단위 | metric label/time range | 개별 flow log |
| 위험 | label cardinality | 로그량 폭증 |

운영 시 exporter를 전체 flow로 오래 켜두는 것은 위험합니다.

권장:

- 먼저 짧은 PoC로 event 형식 확인
- 운영 전에는 `allowList`, `denyList`, `fieldMask`로 범위 축소
- drop/error, 특정 namespace, 특정 service 등으로 필터링
- 로그 보관 기간과 저장 비용 사전 산정

## 현재 파일과 설계 의미

| 파일 | 의미 |
| --- | --- |
| `manifests/30-cni-configuration.yaml` | Calico CNI 뒤에 `cilium-cni`를 추가하는 CNI chaining ConfigMap |
| `manifests/30-cilium-chaining-values.yaml` | 기본 chaining + Hubble Relay/UI values |
| `manifests/15-cilium-hubble-metrics-values.yaml` | chaining 유지 + Hubble metrics + ServiceMonitor + Grafana dashboard values |
| `manifests/15-kube-prometheus-stack-values.yaml` | Prometheus/Grafana 운영형 PoC values |
| `manifests/15-cilium-values-hubble-exporter.yaml` | metrics values를 base로 exporter까지 추가한 생성 values |

values 누적 관계:

```text
30-cilium-chaining-values.yaml
  |
  v
15-cilium-hubble-metrics-values.yaml
  |
  v
15-cilium-values-hubble-exporter.yaml
```

따라서 Grafana metrics를 이미 적용한 뒤 exporter를 추가할 때는 `15-cilium-hubble-metrics-values.yaml`을 base로 사용해야 합니다.

## 현재 설계의 장점

- NKS 기본 Calico를 건드리지 않아 full replacement보다 위험이 낮습니다.
- Pod IPAM과 기본 네트워킹은 NKS 기본 상태를 유지합니다.
- Hubble UI/CLI로 namespace, service, DNS 중심 flow를 볼 수 있습니다.
- Prometheus/Grafana로 운영형 metrics 대시보드를 구성할 수 있습니다.
- exporter로 장기 flow 로그 저장 후보를 검증할 수 있습니다.
- 애플리케이션 Pod에 sidecar를 넣지 않습니다.

## 현재 설계의 한계

- Cilium 완전 대체가 아닙니다.
- Calico 장애나 NKS Calico 변경의 영향을 계속 받습니다.
- Cilium Ingress/Gateway API 기반 전환과는 별개입니다.
- Hubble은 네트워크 flow 중심이며, 애플리케이션 trace/APM 전체를 대체하지 않습니다.
- Grafana metrics는 개별 flow 원문 검색에 적합하지 않습니다.
- exporter는 로그량과 저장 비용 관리가 필요합니다.
- L7 HTTP metrics는 별도 L7 visibility 조건이 필요합니다.

## 진단 관점 분리

문제가 생겼을 때는 아래처럼 계층을 나눠 봅니다.

| 증상 | 우선 확인 계층 |
| --- | --- |
| Pod IP가 안 잡힘 | Calico CNI, IPAM, node 상태 |
| Pod 통신 실패 | Calico VXLAN, kube-proxy, NetworkPolicy |
| Cilium endpoint 없음 | CNI chaining, Cilium agent, 신규 Pod 여부 |
| Hubble flow 안 보임 | Cilium endpoint, Hubble server, Relay |
| Hubble CLI 접속 실패 | port-forward, Relay service, CLI/Relay 버전 |
| Grafana no data | ServiceMonitor, Prometheus target, metrics enabled |
| exporter log 없음 | exporter filePath, Cilium agent restart, flow 발생 여부 |

## 운영 전환 시 추가로 결정할 것

- Hubble UI 접근 방식: port-forward 유지, 내부 Ingress, SSO/VPN 연계
- Prometheus 보관 기간과 storage class
- Hubble metrics label 범위와 cardinality 제한
- exporter 수집 범위: 전체 flow, drop/error, 특정 namespace
- 로그 저장소: Loki, OpenSearch, SIEM
- 로그 수집기: Fluent Bit, Vector, Promtail 등
- Cilium/Hubble CLI 버전 관리
- node scale-out, reboot 이후 Cilium chaining 유지 검증
- 기존 Pod 재시작 정책

## 결론

현재 설계의 본질은 `Calico 기반 네트워킹 유지 + Cilium/Hubble 기반 sidecar-less 네트워크 관측 추가`입니다.

운영형 관측은 세 계층으로 나누는 것이 적절합니다.

| 계층 | 도구 | 목적 |
| --- | --- | --- |
| 즉시 진단 | Hubble CLI/UI | 현재 flow와 drop 확인 |
| 상시 관측 | Prometheus/Grafana | 추세, 알람, 대시보드 |
| 장기 검색 | Hubble exporter + 로그 플랫폼 | 개별 flow 감사/검색 |

이 구조는 서비스메쉬 sidecar 기반 관측보다 애플리케이션 Pod 변경이 적고, NKS 기본 CNI를 유지하므로 PoC 안정성이 높습니다. 다만 Cilium full replacement가 아니므로 Cilium의 모든 네트워크 기능을 운영 전제로 삼으면 안 됩니다.

## 참고 출처

- Cilium generic-veth chaining: https://docs.cilium.io/en/stable/installation/cni-chaining-generic-veth/
- Cilium Hubble setup: https://docs.cilium.io/en/stable/observability/hubble/setup/
- Cilium Prometheus/Grafana: https://docs.cilium.io/en/stable/observability/grafana/
- Cilium metrics: https://docs.cilium.io/en/stable/observability/metrics/
- Cilium Hubble exporter: https://docs.cilium.io/en/stable/observability/hubble/configuration/export/
- Calico overlay networking: https://docs.tigera.io/calico/latest/networking/configuring/vxlan-ipip
- Prometheus Operator introduction: https://prometheus-operator.dev/docs/getting-started/introduction/
