# Cilium Service Mesh 개념 정리

- 목적: Cilium Service Mesh의 구성요소와 NKS Calico chaining 환경에서의 제약을 이론적으로 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-05-13

## 먼저 결론

현재 클러스터의 `Calico 유지 + Cilium generic-veth chaining + Hubble` 구성은 네트워크 flow 관측 기반으로는 유효합니다.
하지만 검증 결과 Cilium Service Mesh에 필요한 Envoy L7 redirect datapath는 성립하지 않았습니다.

핵심 차이는 아래와 같습니다.

| 구분 | 현재 Hubble 관측 구성 | Cilium Service Mesh 구성 |
| --- | --- | --- |
| 주 목적 | 네트워크 flow 관측 | L7 트래픽 처리, 라우팅, 보안, 관측 |
| Primary CNI | Calico 유지 | Cilium primary 권장 |
| Cilium 역할 | chained CNI + Hubble 관측 | primary CNI + Service datapath/L7 proxy 연계 |
| `kubeProxyReplacement` | `false` | `true` 필요 |
| Envoy | 기동 중이지만 핵심 경로 아님 | Ingress/Gateway/GAMMA/L7 경로의 핵심 |
| Gateway API CRD | 없음 | 필요 |
| 위험 범위 | 낮음 | Service 처리 경로 영향 |

따라서 현재 NKS Calico chaining 환경은 Hubble 관측에는 적합하지만, Cilium Service Mesh 운영 환경으로 보지는 않습니다.

## 기존 이론 문서와의 관계

기존 이론 문서:

- `docs/01-observability-architecture-concepts.md`

기존 문서는 아래를 설명합니다.

- Calico VXLAN이 primary CNI인 이유
- Cilium generic-veth chaining의 의미
- Hubble UI/CLI/metrics/exporter 관측 구조
- Prometheus/Grafana 연계

이 문서는 그 다음 단계인 Cilium Service Mesh를 설명합니다.

즉, 기존 문서가 “sidecar-less 네트워크 관측” 이론이라면, 이 문서는 “Cilium이 L7 트래픽 처리와 서비스 간 라우팅에 관여할 때의 구조”를 다룹니다.

## Service Mesh 일반 개념

Service Mesh는 애플리케이션 코드에 직접 넣던 네트워크 기능을 인프라 계층으로 빼는 방식입니다.

대표 기능:

- 서비스 간 연결성
- L7 트래픽 관리
- identity 기반 보안
- 관측과 추적
- 애플리케이션 변경 최소화

전통적인 Service Mesh는 보통 각 Pod에 sidecar proxy를 넣습니다.

```text
Pod A
  app container
  sidecar proxy
        |
        v
Pod B
  sidecar proxy
  app container
```

이 방식은 서비스별로 proxy가 붙기 때문에 세밀한 제어가 가능하지만, Pod 수만큼 sidecar가 늘어납니다.
CPU/메모리 오버헤드, lifecycle 관리, 장애 지점 증가가 생깁니다.

## Cilium Service Mesh의 기본 접근

Cilium Service Mesh는 Cilium의 eBPF datapath와 Envoy proxy를 결합합니다.

핵심 구조:

```text
L3/L4 path
  Cilium eBPF datapath

L7 path
  Cilium Envoy proxy

Control
  Kubernetes Ingress
  Gateway API
  GAMMA HTTPRoute
  CiliumEnvoyConfig

Observability
  Hubble
  Prometheus/Grafana
```

Cilium은 IP/TCP/UDP 같은 네트워크 계층 처리는 eBPF로 처리하고, HTTP/gRPC/DNS 같은 애플리케이션 계층 처리는 Envoy를 사용합니다.

중요한 점은 “sidecar-less”입니다.
Cilium Service Mesh는 일반적으로 애플리케이션 Pod마다 proxy sidecar를 넣는 방식이 아니라, 노드 단의 Cilium/Envoy 구성으로 트래픽을 처리합니다.

## 현재 NKS 구성과의 관계

현재 클러스터는 아래 구조입니다.

```text
NKS worker node
  |
  +-- Calico
  |     - Pod IPAM
  |     - Pod veth 생성
  |     - VXLAN overlay
  |
  +-- Cilium chaining
  |     - Calico가 만든 veth에 eBPF program attach
  |     - endpoint/identity/flow 관측
  |
  +-- Cilium Envoy
        - 현재는 Ready
        - Service Mesh 활성화 시 L7 proxy 경로 담당
```

Service Mesh를 검증한다고 해서 Calico를 제거하는 것은 아닙니다.
하지만 `kubeProxyReplacement=true`를 켜면 Cilium이 Kubernetes Service 처리 경로에 더 깊게 관여합니다.

그래서 아래 두 문장을 구분해야 합니다.

| 문장 | 의미 |
| --- | --- |
| Calico를 Cilium으로 full replacement한다 | Primary CNI를 바꾸는 큰 전환. 현재 프로젝트에서는 실패로 종결 |
| Cilium Service Mesh를 켠다 | Calico는 유지하되 Cilium의 Service/L7 처리 기능을 활성화 |

현재 우리가 검토하는 것은 두 번째입니다.

## 왜 `kubeProxyReplacement=true`가 중요한가

Cilium Gateway API, Ingress, GAMMA, L7 Traffic Management는 Cilium이 Service 처리 경로를 제어한다는 전제가 있습니다.

현재 값:

```yaml
kubeProxyReplacement: "false"
```

Service Mesh 후보 값:

```yaml
kubeProxyReplacement: "true"
gatewayAPI:
  enabled: true
l7Proxy: true
```

이 변경은 Cilium이 kube-proxy 역할을 일부 또는 전체 대체하도록 만드는 방향입니다.
따라서 단순 관측 기능을 켜는 것보다 영향 범위가 큽니다.

확인해야 할 것:

- ClusterIP Service 통신 유지
- CoreDNS 정상
- metrics API 정상
- NodePort/LoadBalancer 동작
- 기존 kube-system addon 영향
- 신규 Pod와 기존 Pod의 Service 통신 차이

## Cilium Envoy의 역할

Envoy는 Cilium Service Mesh의 L7 proxy입니다.

역할:

- HTTP/gRPC 요청 라우팅
- header/path 기반 routing
- traffic split
- TLS termination 또는 passthrough 일부 처리
- L7 policy enforcement point
- Hubble HTTP metrics 생성에 필요한 L7 visibility 제공

현재 클러스터에는 `cilium-envoy` DaemonSet이 이미 Ready입니다.
하지만 현재 Hubble 관측 구성에서는 모든 트래픽이 Envoy를 통과한다는 의미가 아닙니다.

Service Mesh를 활성화하면 Gateway API, Ingress, GAMMA, CiliumEnvoyConfig가 만든 L7 경로에서 Envoy가 실제 데이터 경로에 들어옵니다.

## Gateway API

Gateway API는 Kubernetes Ingress의 후속 표준 성격을 가진 API입니다.

핵심 리소스:

| 리소스 | 역할 |
| --- | --- |
| `GatewayClass` | 어떤 controller가 Gateway를 처리할지 정의 |
| `Gateway` | listener, port, hostname, TLS 같은 진입점 정의 |
| `HTTPRoute` | HTTP routing rule 정의 |
| `GRPCRoute` | gRPC routing rule 정의 |
| `TLSRoute` | TLS passthrough routing. experimental |
| `ReferenceGrant` | namespace 간 참조 허용 |

Cilium Gateway API의 기본 흐름:

```text
External client
  |
  v
LoadBalancer Service or hostNetwork listener
  |
  v
Cilium eBPF
  |
  v
Cilium Envoy
  |
  v
Kubernetes Service
  |
  v
Backend Pod
```

Cilium Gateway API는 기본적으로 LoadBalancer Service를 만듭니다.
LoadBalancer를 사용할 수 없는 환경에서는 hostNetwork mode를 선택할 수 있습니다.

현재 NKS에서는 LoadBalancer 동작을 별도로 검증해야 합니다.
hostNetwork는 모든 Cilium node interface에 listener가 노출될 수 있으므로 보안그룹과 포트 충돌을 별도로 봐야 합니다.

## Ingress와 Gateway API 차이

둘 다 north-south 트래픽, 즉 클러스터 밖에서 안으로 들어오는 트래픽을 다룹니다.

| 항목 | Ingress | Gateway API |
| --- | --- | --- |
| 성격 | 오래된 표준 API | Ingress 후속 표준 |
| 리소스 | `Ingress` 중심 | `GatewayClass`, `Gateway`, `HTTPRoute` 분리 |
| 역할 분리 | 약함 | 인프라 팀/Gateway 소유자/앱 팀 분리 가능 |
| 표현력 | 기본 HTTP/TLS 중심 | HTTP/gRPC/TLS, traffic split 등 확장성 높음 |
| Cilium PoC 우선순위 | 보조 후보 | 주 후보 |

현재 프로젝트에서는 Gateway API를 우선 후보로 둡니다.
이유는 고객 요청이 “Service Mesh”에 가까울수록 Gateway API/GAMMA가 장기 구조와 더 잘 맞기 때문입니다.

## GAMMA

GAMMA는 Gateway API를 Service Mesh 용도로 확장해 쓰는 흐름입니다.

일반 Gateway API:

```text
HTTPRoute
  parentRef: Gateway

외부 -> Gateway -> Service -> Pod
```

GAMMA:

```text
HTTPRoute
  parentRef: Service

Pod -> Service -> Envoy L7 routing -> Backend Pod
```

즉, GAMMA는 north-south가 아니라 east-west, 서비스 간 내부 호출을 다룹니다.

Cilium에서 GAMMA의 의미:

- Service를 parentRef로 하는 `HTTPRoute`를 사용
- 특정 Service로 들어가는 L7 트래픽을 Envoy로 처리
- 앱은 일반 Service를 호출하지만, Cilium이 L7 routing을 개입

주의:

- Cilium 문서 기준 GAMMA는 experimental 성격이 남아 있습니다.
- Cilium은 producer route 중심으로 지원합니다.
- consumer route는 현재 주요 제약입니다.
- 따라서 운영 기능으로 확정하기 전 PoC 결과가 필요합니다.

## L7 Traffic Management

Cilium은 Gateway API/GAMMA 외에도 `CiliumEnvoyConfig`와 `CiliumClusterwideEnvoyConfig`로 Envoy 설정을 직접 다룰 수 있습니다.

가능한 일:

- path rewrite
- header 기반 routing
- traffic shifting
- circuit breaking
- service별 L7 load balancing

하지만 이 방식은 고급 기능입니다.
Gateway API보다 검증과 충돌 관리가 어렵습니다.

운영상 원칙:

- 먼저 Gateway API/GAMMA로 가능한지 본다.
- 직접 `CiliumEnvoyConfig`를 쓰는 것은 마지막 수단으로 둔다.
- Cilium이 자동 생성한 Envoy 설정과 충돌하지 않는지 검증한다.

## mTLS와 Mutual Authentication

Cilium에는 mutual authentication 기능이 있습니다.
다만 현재 문서 기준 beta이며, 기능이 아직 완성형이라고 전제하면 안 됩니다.

일반 mTLS:

```text
Client service <--- TLS mutual auth ---> Server service
```

Cilium mutual authentication의 방향:

- service-to-service identity 확인
- SPIFFE/SPIRE 기반 identity 체계 사용
- Cilium security identity와 연계
- 암호화 기능과 함께 검토 필요

현재 프로젝트 판단:

- 이번 Service Mesh 1차 PoC 범위에는 넣지 않습니다.
- Gateway API/GAMMA/L7 관측이 먼저입니다.
- 고객이 mTLS를 명시 요구하면 별도 보안 트랙으로 분리합니다.

## Hubble과 Service Mesh 관측

Hubble은 Service Mesh 활성화 이후에도 핵심 관측 도구입니다.

기존 Hubble 관측:

```text
Pod -> Service -> Pod
  L3/L4 flow 중심
```

Service Mesh 이후 관측:

```text
Client -> Gateway/Envoy -> Service -> Pod
Client Pod -> Service/GAMMA/Envoy -> Backend Pod
  L3/L4 + 일부 L7 HTTP/gRPC flow
```

기대할 수 있는 것:

- Gateway를 통과한 요청 flow
- source/destination namespace/workload
- HTTP method/path/status code 기반 metrics
- drop/error verdict
- DNS와 TCP 흐름

주의:

- 모든 트래픽이 자동으로 L7로 보이는 것은 아닙니다.
- L7 proxy 경로를 통과하거나 L7 visibility 조건을 만족해야 HTTP metrics가 의미 있게 나옵니다.
- 암호화된 TLS payload는 termination/passthrough 구조에 따라 관측 가능 범위가 달라집니다.

## 현재 환경에서의 목표 아키텍처

1차 PoC 목표 구조:

```text
External client
  |
  v
NKS LoadBalancer or hostNetwork
  |
  v
Cilium Gateway API
  |
  v
Cilium Envoy per-node proxy
  |
  v
Service web-v1 / web-v2
  |
  v
Pod web-v1 / web-v2

Observability:
  Cilium/Hubble metrics -> Prometheus -> Grafana
  Hubble Relay -> Hubble CLI/UI
  Hubble exporter -> flow log
```

GAMMA PoC 목표 구조:

```text
curl Pod
  |
  v
Service web
  |
  v
HTTPRoute parentRef=Service
  |
  v
Cilium Envoy
  |
  v
web-v1 / web-v2 backend
```

## 적용 전 반드시 이해할 영향

`kubeProxyReplacement=true`는 이론상 Cilium Service Mesh 전제지만, 현재 NKS Calico chaining 환경에서는 미검증입니다.

따라서 다음 질문에 답해야 합니다.

| 질문 | 확인 방법 |
| --- | --- |
| ClusterIP Service가 계속 정상인가 | CoreDNS, Kubernetes Service, 샘플 Service 호출 |
| metrics API가 유지되는가 | `v1beta1.metrics.k8s.io` 확인 |
| Cilium endpoint가 정상 생성되는가 | 신규 Pod와 `cilium status` 확인 |
| GatewayClass가 생성되는가 | `kubectl get gatewayclass` |
| Gateway가 Programmed 되는가 | Gateway status 확인 |
| LoadBalancer가 붙는가 | Gateway가 만든 Service 확인 |
| hostNetwork가 필요한가 | LB 미할당 시 검토 |
| Hubble에서 L7 flow가 보이는가 | Gateway/GAMMA 요청 후 Hubble 확인 |

## 가능한 검증 순서

```text
1. 현재 Cilium/Hubble 정상 확인
2. Gateway API CRD 설치
3. Helm dry-run
4. Cilium Gateway API 활성화
5. Cilium/CoreDNS/metrics API 확인
6. Gateway HTTP 샘플
7. Hubble/Grafana 관측
8. GAMMA HTTPRoute 샘플
9. 결과에 따라 유지 또는 롤백
```

이 순서를 지키는 이유는 장애 지점을 좁히기 위해서입니다.
CRD 문제, Cilium 설정 문제, LoadBalancer 문제, Gateway route 문제, backend 문제를 한 번에 섞지 않습니다.

## 운영 판단 기준

Service Mesh PoC가 성공해도 바로 운영 적용으로 보기는 어렵습니다.

운영 전 추가 기준:

- node reboot 후 Gateway listener 복구
- node scale-out 후 Cilium/Gateway/Envoy 동작
- LoadBalancer 장애 시 동작
- hostNetwork 사용 시 보안그룹과 포트 충돌
- backend 배포 중 connection 영향
- HTTPRoute 변경 시 무중단성
- Hubble metrics cardinality
- Envoy 로그와 Cilium agent 로그 수집
- mTLS 요구 여부
- 기존 Ingress Controller와 공존 여부

## 현재 프로젝트의 판단

현재 프로젝트에서 Cilium Service Mesh는 “현재 NKS Calico chaining 환경에서는 불가”로 판정합니다.

판단:

- 같은 NKS Calico chaining 클러스터에서 Gateway API/GAMMA 제어면 PoC는 가능했지만 실제 L7 datapath는 실패
- Calico를 primary CNI로 유지하는 조건에서는 Cilium Service Mesh를 운영 수준으로 구성하지 않음
- `kubeProxyReplacement=true` 적용 후 기본 Service 통신은 유지됐지만 Envoy L7 redirect가 timeout
- Gateway API/Ingress/GAMMA/L7 Policy는 현재 운영 범위에서 제외
- mTLS는 별도 후속 트랙

따라서 현재 트랙은 Cilium Service Mesh 도입 트랙이 아니라, 실패 근거와 대안 검토를 남기는 트랙으로 관리합니다.
Cilium Service Mesh 표준 구축 절차는 Cilium primary CNI가 가능한 별도 환경 기준으로 `01-cilium-service-mesh-runbook.md`에 분리합니다.

## 용어 요약

| 용어 | 의미 |
| --- | --- |
| CNI chaining | 기존 CNI 뒤에 Cilium CNI를 이어 붙이는 방식 |
| primary CNI | Pod IPAM과 기본 Pod 네트워크를 소유하는 CNI. 현재는 Calico |
| kube-proxy replacement | Kubernetes Service 처리 경로를 Cilium eBPF로 대체하는 기능 |
| Envoy | Cilium이 L7 트래픽 처리에 사용하는 proxy |
| Gateway API | Ingress 후속 성격의 Kubernetes service networking API |
| GAMMA | Gateway API를 Service Mesh 내부 트래픽에 쓰는 방식 |
| north-south | 클러스터 외부와 내부 간 트래픽 |
| east-west | 클러스터 내부 서비스 간 트래픽 |
| L7 | HTTP/gRPC/DNS 등 애플리케이션 계층 |
| Hubble | Cilium flow observability 계층 |

## 참고 출처

- Cilium Service Mesh: https://docs.cilium.io/en/stable/network/servicemesh/
- Cilium Gateway API: https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/
- Cilium GAMMA: https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gamma/
- Cilium Ingress: https://docs.cilium.io/en/stable/network/servicemesh/ingress/
- Cilium L7 Traffic Management: https://docs.cilium.io/en/stable/network/servicemesh/l7-traffic-management/
- Cilium Mutual Authentication: https://docs.cilium.io/en/stable/network/servicemesh/mutual-authentication/mutual-authentication/
- Cilium CNI Chaining: https://docs.cilium.io/en/stable/installation/cni-chaining/
