# Cilium chaining 관측 구성과 Istio Ambient 운영 무게 비교

- 목적: 현재 NKS Calico 유지 + Cilium chaining + Hubble 관측 구성과 Istio Ambient + Kiali 도입안을 운영 무게 관점에서 비교합니다.
- 상태: draft
- 마지막 갱신: 2026-05-13

## 결론

현재 요구가 아래 수준이면 `Calico 유지 + Cilium generic-veth chaining + Hubble` 설계가 운영 무게 측면에서 더 적합합니다.

- sidecar-less 네트워크 flow 관측
- Pod 간 L3/L4 통신 흐름 관측
- 기존 Calico primary CNI 유지
- 외부 진입은 NKS LoadBalancer 또는 별도 Ingress Controller로 처리

반대로 아래 요구가 명확하면 Cilium chaining이 아니라 Istio Ambient + Kiali 같은 별도 Service Mesh가 기능적으로 더 적합합니다.

- mesh identity 기반 mTLS
- L7 authorization policy
- waypoint 기반 표준 Istio traffic management
- Kiali 중심 서비스 그래프
- Istio 생태계 기반 운영 표준화
- east-west L7 라우팅 운영 적용

다만 현재 2-node NKS PoC 클러스터 기준으로 Istio Ambient + Kiali는 기본 request만으로도 `1110m CPU / 3336Mi Memory`를 예약합니다.
waypoint 1개와 Istio ingress gateway 1개까지 포함하면 최소 `1310m CPU / 3592Mi Memory`가 됩니다.

현재 Cilium/Hubble 추가 운영분의 실측 사용량은 `71m CPU / 482Mi Memory`입니다.
즉, 현재 관측된 idle/저부하 기준으로는 Istio Ambient + Kiali 기본 예약량이 Cilium/Hubble 추가분보다 CPU 약 `15.6배`, Memory 약 `6.9배` 큽니다.
waypoint와 Istio ingress gateway까지 포함하면 CPU 약 `18.5배`, Memory 약 `7.5배`입니다.

## 산정 기준

### 현재 클러스터

2026-05-13 read-only 조회 기준입니다.

| 항목 | 값 |
| --- | --- |
| 클러스터 | `ta-sgh-vxlan-sidecarless-cls` |
| Kubernetes | `v1.33.4` |
| 노드 | worker 2대 |
| Primary CNI | NKS 기본 Calico VXLAN |
| Cilium | `1.19.3`, `generic-veth` chaining |
| Hubble | Relay/UI/metrics 활성화 |
| Cilium Service Mesh 검증 | Gateway/Ingress/GAMMA/L7 Policy 모두 운영 수준 실패 |

### 비교 대상

Istio/Kiali는 현재 클러스터에 설치되어 있지 않으므로 공식 Helm chart 기본 manifest를 렌더링해서 request 값을 산정했습니다.

| 항목 | 기준 |
| --- | --- |
| Istio chart | `1.29.2` |
| Kiali chart | `2.26.0` |
| 기준 명령 | `helm show chart`, `helm template`, `kubectl apply --dry-run=client -o json` |
| 산정 범위 | `istiod`, `istio-cni`, `ztunnel`, `kiali`, 선택 항목으로 waypoint/gateway |

Prometheus, Tempo, Jaeger, Grafana 등 별도 관측 저장소는 산정에서 제외했습니다.
Kiali 서비스 그래프와 추적을 운영 수준으로 쓰려면 Prometheus와 tracing backend가 필요하므로, 실제 운영 무게는 이 문서의 Istio 최소값보다 커질 수 있습니다.

## 현재 구성 실측값

`kubectl top pod -A` 기준입니다.

| 구성요소 | Pod 수 | CPU 사용량 | Memory 사용량 | 비고 |
| --- | ---: | ---: | ---: | --- |
| Calico | 4 | 76m | 424Mi | 기존 baseline |
| Cilium agent | 2 | 57m | 376Mi | chaining + Hubble datapath |
| Cilium Envoy | 2 | 8m | 33Mi | Gateway/GAMMA/Ingress L7 proxy |
| Cilium operator | 1 | 4m | 45Mi | control plane |
| Hubble Relay | 1 | 1m | 16Mi | flow query |
| Hubble UI | 1 | 1m | 12Mi | frontend/backend 합산 |
| Cilium/Hubble 소계 | 7 | 71m | 482Mi | Calico 위에 추가된 운영분 |
| Calico + Cilium/Hubble 합계 | 11 | 147m | 906Mi | 현재 네트워크/관측 전체 |

노드 사용량은 아래와 같습니다.

| 노드 | CPU | Memory |
| --- | ---: | ---: |
| `worker-node-3` | 117m, 5% | 1123Mi, 57% |
| `worker-node-4` | 101m, 5% | 1076Mi, 54% |

주의할 점은 현재 Cilium/Hubble workload에는 request/limit이 명시되어 있지 않다는 점입니다.
실측 사용량은 낮지만 운영 환경에서는 최소 request를 별도로 잡아야 스케줄링과 eviction 기준이 명확해집니다.

## Istio Ambient + Kiali 기본 request

공식 Helm chart 기본값 기준입니다.

| 구성요소 | 배치 방식 | 개수 | Pod당 request | 2-node 합산 |
| --- | --- | ---: | ---: | ---: |
| `istiod` | Deployment | 1 | 500m / 2048Mi | 500m / 2048Mi |
| `istio-cni-node` | DaemonSet | 2 | 100m / 100Mi | 200m / 200Mi |
| `ztunnel` | DaemonSet | 2 | 200m / 512Mi | 400m / 1024Mi |
| `kiali` | Deployment | 1 | 10m / 64Mi | 10m / 64Mi |
| 합계 | - | 6 | - | 1110m / 3336Mi |

선택 기능을 포함하면 아래가 추가됩니다.

| 추가 구성요소 | 용도 | 기본 request |
| --- | --- | ---: |
| waypoint 1개 | L7 east-west traffic management | 100m / 128Mi |
| Istio ingress gateway 1개 | north-south 진입 | 100m / 128Mi |

따라서 최소 운영 단위는 아래처럼 나뉩니다.

| 시나리오 | Pod 수 | 기본 request |
| --- | ---: | ---: |
| Ambient + Kiali | 6 | 1110m / 3336Mi |
| Ambient + Kiali + waypoint 1개 | 7 | 1210m / 3464Mi |
| Ambient + Kiali + waypoint 1개 + ingress gateway 1개 | 8 | 1310m / 3592Mi |

## 정량 비교

### Calico 유지 baseline 위 추가분 기준

현재 클러스터는 Calico를 primary CNI로 유지합니다.
따라서 공정한 비교는 Calico 위에 추가되는 운영분을 보는 것입니다.

| 항목 | Cilium chaining + Hubble | Istio Ambient + Kiali | 차이 |
| --- | ---: | ---: | ---: |
| 최소 Pod 수 | 7 | 6 | Istio가 1개 적음 |
| CPU | 71m 실측 | 1110m request | Istio가 약 15.6배 큼 |
| Memory | 482Mi 실측 | 3336Mi request | Istio가 약 6.9배 큼 |
| L7 east-west 포함 | 현재 구성에서 실패 | waypoint 추가 필요 | Istio +100m / +128Mi |
| north-south mesh 포함 | 현재 구성에서 실패 | gateway 추가 필요 | Istio +100m / +128Mi |

### north-south까지 포함한 최소값

| 항목 | Cilium current track | Istio Ambient track |
| --- | ---: | ---: |
| CNI baseline | Calico 유지 | Calico 유지 가능 |
| Mesh/관측 추가분 | 71m / 482Mi 실측 | 1310m / 3592Mi request |
| 외부 진입 | 별도 Ingress/LB 권장 | Istio gateway 가능 |
| 현재 NKS 검증 결과 | Cilium Gateway/Ingress 불안정 | 별도 PoC 필요 |

이 수치는 Cilium은 실측 사용량, Istio는 chart 기본 request라는 차이가 있습니다.
하지만 운영자는 request 기준으로 노드 용량을 확보해야 하므로, 운영 계획에서는 Istio 쪽 수치를 최소 예약량으로 보는 것이 맞습니다.

## 기능 비교

| 항목 | Cilium chaining + Hubble | Istio Ambient + Kiali |
| --- | --- | --- |
| 기본 목적 | eBPF 기반 flow 관측 | service mesh 표준 기능 |
| sidecar | 없음 | 없음 |
| L4 관측 | 강함 | 가능 |
| L7 라우팅 | Gateway/GAMMA/L7 Policy 모두 현재 클러스터에서 실패 | waypoint 기반으로 강함 |
| mTLS | 현재 설계 범위 밖 | ambient 핵심 기능 |
| authorization policy | 제한적 | 강함 |
| 서비스 그래프 | Hubble/Grafana 중심, trace 아님 | Kiali 중심, Prometheus 필요 |
| distributed tracing | 별도 OpenTelemetry/Tempo/Jaeger 필요 | 별도 OpenTelemetry/Tempo/Jaeger 필요 |
| 외부 진입 | 현재 Cilium Gateway/Ingress는 운영 판정 불가 | Istio ingress gateway로 설계 가능 |
| 운영 복잡도 | 낮음에서 중간 | 중간에서 높음 |

## 데이터 경로 비교

### Cilium chaining + Hubble

Flow 관측만 사용할 때는 애플리케이션 트래픽에 별도 proxy hop을 강제하지 않습니다.
Cilium eBPF/Hubble이 flow를 관측합니다.

GAMMA HTTPRoute와 HTTP L7 Policy는 Cilium Envoy가 L7 처리에 관여해야 합니다.
검증 결과 현재 NKS Calico chaining 환경에서는 이 경로가 timeout으로 실패했습니다.

현재 검증 결과:

- `HTTPRoute parentRef=Service` 기반 GAMMA east-west 라우팅은 상태상 Accepted/ResolvedRefs True였지만, 실제 parent Service 요청 timeout이 재현됐습니다.
- HTTP L7 CiliumNetworkPolicy도 적용 시 timeout, 제거 시 복구가 확인됐습니다.
- Cilium Gateway API/Ingress 기반 north-south 진입은 LoadBalancer, NodePort, hostNetwork 방식 모두 운영 안정성을 확보하지 못했습니다.
- 원인은 보안그룹 단독 문제로 보기 어렵고, Calico primary + Cilium chaining + Cilium Envoy north-south 경로의 조합 문제로 판단합니다.

### Istio Ambient + Kiali

Ambient mesh에 포함된 namespace/workload는 ztunnel을 통해 L4 mesh datapath에 들어갑니다.
L7 정책, 라우팅, 세밀한 telemetry가 필요하면 waypoint proxy를 추가합니다.

운영 관점에서는 다음 특성이 있습니다.

- ztunnel은 DaemonSet으로 모든 노드에 상주합니다.
- waypoint는 L7 기능이 필요한 namespace, service account, service 단위로 증가합니다.
- Kiali는 자체 datapath 구성요소는 아니지만 Prometheus 기반 telemetry에 의존합니다.
- tracing까지 요구하면 Tempo 또는 Jaeger 같은 backend가 추가됩니다.

## 운영 무게 평가

점수는 현재 프로젝트 목적 기준입니다.
낮을수록 가볍고, 높을수록 무겁습니다.

| 평가 항목 | Cilium chaining + Hubble | Istio Ambient + Kiali | 판단 |
| --- | ---: | ---: | --- |
| 리소스 예약 부담 | 2 | 5 | Istio 기본 request가 큼 |
| 구성요소 수 | 3 | 4 | Istio는 waypoint/관측 backend가 늘어남 |
| 데이터패스 변경 범위 | 3 | 4 | Istio는 mesh 포함 트래픽이 ztunnel 경유 |
| 기능 완성도 | 3 | 5 | Istio가 service mesh 기능 우위 |
| 현재 NKS 적합성 | 4 | 3 | Cilium은 관측 한정 적합, Istio는 별도 검증 필요 |
| 운영 학습/장애 대응 부담 | 3 | 5 | Istio 정책/telemetry/control plane 이해 필요 |
| 고객 설명 용이성 | 3 | 4 | Istio/Kiali가 인지도는 높지만 운영 비용 설명 필요 |

요약하면 Cilium 설계는 가볍지만 기능 범위가 제한적입니다.
Istio Ambient 설계는 기능적으로 더 완성형이지만, 현재 클러스터 규모에서는 운영 무게가 큽니다.

## 권장 판단

### 현재 요구가 flow 관측 + service mesh 도입 근거 확보인 경우

권장안:

- Calico primary CNI 유지
- Cilium generic-veth chaining 유지
- Hubble/Grafana 기반 flow 관측 운영화
- Cilium GAMMA/L7 Policy는 현재 검증 결과 기준 운영 제외
- 외부 진입은 NKS LoadBalancer 또는 별도 Ingress Controller로 분리

이 방식은 현재 검증 결과와 가장 잘 맞습니다.
운영 무게도 가장 낮습니다.

### 고객이 완전한 service mesh 기능을 요구하는 경우

권장안:

- Istio Ambient + Kiali를 별도 트랙으로 PoC
- 최소 3-node 이상 또는 현재 노드 증설 후 검증
- resource request를 운영 기준으로 재조정
- Prometheus, tracing backend 포함 총 비용 재산정
- waypoint 수 증가에 따른 리소스와 장애 범위 산정

이 경우 Cilium chaining + Hubble은 Service Mesh 대체안으로 보기 어렵습니다.
특히 mTLS, L7 authorization, Kiali 서비스 그래프가 요구사항이면 Istio 계열이 더 적합합니다.

### 외부 진입까지 mesh 요구가 있는 경우

현재 Cilium Gateway API/Ingress는 이 NKS Calico chaining 환경에서 운영 수준으로 판정하지 않습니다.
따라서 외부 진입까지 service mesh 요구가 있으면 아래 중 하나로 설계를 나누는 것이 안전합니다.

| 방안 | 판단 |
| --- | --- |
| Cilium Gateway/Ingress 계속 사용 | 현재 검증 결과 기준 비권장 |
| 별도 Ingress/LB + Cilium GAMMA east-west | 현재 검증 결과 기준 비권장 |
| Istio ingress gateway + Ambient | 기능상 후보, 별도 PoC 필요 |
| Cilium full replacement 재시도 | 이전 Track A 실패 이력으로 비권장 |

## 잔여 리스크

- Cilium/Hubble 현재 workload에 request/limit이 없어 운영 스케줄링 기준을 별도로 정해야 합니다.
- Cilium GAMMA east-west와 HTTP L7 Policy는 timeout이 재현되어 운영 적용에서 제외합니다.
- Cilium Gateway/Ingress north-south는 현재 구성에서 운영 안정성을 확보하지 못했습니다.
- Istio Ambient는 이 클러스터에 실제 설치 검증을 수행하지 않았습니다.
- Kiali 운영에는 Prometheus가 필요하고, tracing까지 요구하면 별도 backend가 필요합니다.
- 이 문서의 정량 비교는 idle/저부하 실측과 Helm 기본 request 비교입니다. 실제 성능 비교는 `fortio`, `wrk`, `k6` 등으로 별도 부하 테스트가 필요합니다.

## 참고 근거

- Cilium Service Mesh: <https://docs.cilium.io/en/stable/network/servicemesh/>
- Cilium Gateway API GAMMA: <https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gamma.html>
- Istio Ambient overview: <https://istio.io/latest/docs/ambient/overview/>
- Istio waypoint: <https://istio.io/latest/docs/ambient/usage/waypoint/>
- Kiali Prometheus integration: <https://kiali.io/docs/configuration/p8s-jaeger-grafana/prometheus/>
