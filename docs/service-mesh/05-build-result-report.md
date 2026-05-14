# Calico Chaining 기반 Cilium Service Mesh 검증 보고서

- 목적: NKS 기본 Calico VXLAN 환경에서 Cilium `generic-veth` chaining으로 Cilium Service Mesh를 구성할 수 있는지 검증한 결과와 실패 원인을 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-05-14

## 최종 결론

현재 NKS `Calico primary CNI + Cilium generic-veth chaining` 구조에서는 **Cilium Service Mesh를 운영 수준으로 구성할 수 없습니다.**

가능한 범위는 아래입니다.

- Calico primary CNI 유지
- Cilium generic-veth chaining 유지
- Hubble 기반 L3/L4 flow 관측
- 일반 ClusterIP/NodePort 통신
- 일부 Cilium eBPF 기능과 기본 정책 검증

불가능 또는 운영 적용 제외 범위는 아래입니다.

- Cilium Gateway API 기반 north-south Service Mesh
- Cilium Ingress 기반 north-south Service Mesh
- Cilium GAMMA 기반 east-west Service Mesh
- Cilium HTTP L7 Policy 기반 Envoy proxy 처리
- Hubble만으로 사용자 요청 전체 distributed trace 구성

핵심 실패 지점은 `kube-proxy` 존재 여부가 아니라 **Cilium eBPF 정책 redirect 이후 Envoy listener로 전달되는 구간**입니다.
Cilium Service Mesh는 Cilium eBPF가 Service 트래픽을 처리하고 Envoy로 넘겨야 하는데, 현재 chaining 구조에서는 eBPF가 redirect verdict를 내린 뒤 실제 Envoy listener가 연결을 수락하지 못했습니다.

## 용어 정리

### 기존 NKS 기본 구조

```text
Pod 네트워크/IPAM/VXLAN: Calico
Service VIP 처리: kube-proxy
L7 Service Mesh: 없음
```

### Cilium chaining 관측 구조

```text
Pod 네트워크/IPAM/VXLAN: Calico
Service VIP 처리: kube-proxy 또는 일부 Cilium KPR
관측/일부 eBPF 기능: Cilium + Hubble
```

### Cilium Service Mesh에 필요한 구조

```text
Service VIP 처리: Cilium eBPF kube-proxy replacement
L7 처리: Cilium Envoy
트래픽 전달: Cilium eBPF redirect 또는 TPROXY
```

Envoy가 kube-proxy를 대체하는 것이 아닙니다.
kube-proxy 역할은 Cilium eBPF가 대체하고, Envoy는 HTTP/gRPC 같은 L7 처리를 담당합니다.

## 환경

| 항목 | 값 |
| --- | --- |
| 클러스터 | `ta-sgh-vxlan-sidecarless-cls` |
| Kubernetes | `v1.33.4` |
| 노드 | worker 2대 |
| Primary CNI | NKS 기본 Calico VXLAN |
| Calico | `v3.30.2` |
| Cilium | `1.19.3` |
| Cilium 설치 방식 | `generic-veth` chaining |
| Cilium Envoy | `cilium-envoy` DaemonSet |
| Hubble | Relay/UI/metrics 활성화 |
| Gateway API | standard CRD 설치 |

## 공식 문서상 전제

공식 문서 기준으로 확인한 전제는 아래입니다.

| 기능 | 공식 전제 | 현재 검증 의미 |
| --- | --- | --- |
| CNI chaining | 기본 연결성과 IPAM은 non-Cilium CNI가 담당하고, Cilium은 eBPF 기능을 추가 | Calico가 primary datapath를 계속 소유 |
| Calico chaining | Calico 위에 Cilium chaining 가능, 단 L7 Policy 같은 고급 기능은 제한 가능 | Hubble/L3/L4는 가능하지만 L7 Service Mesh는 위험 |
| Gateway API | `kubeProxyReplacement=true`, `l7Proxy=true`, Gateway API CRD 필요 | Helm 설정과 CRD는 적용 가능 |
| GAMMA | Service parentRef HTTPRoute를 통해 Service 트래픽을 per-node Envoy로 전달 | 리소스 생성은 가능하나 실제 L7 경로가 실패 |
| L7 traffic management | `kubeProxyReplacement=true` 필요, CiliumEnvoyConfig/Envoy 사용 | HTTP L7 Policy 테스트에서 timeout |

## 수행한 검증

### 1. 기본 Cilium chaining 상태

확인 결과:

- `calico-node` Ready
- `cilium` Ready
- `cilium-envoy` Ready
- `hubble-relay`, `hubble-ui` Ready
- `cni-chaining-mode=generic-veth`
- `KubeProxyReplacement=True`까지 활성화 가능

판정:

- Cilium 자체가 고장난 상태는 아닙니다.
- Calico 위에 Cilium chaining과 Hubble 관측을 올리는 구성은 유지 가능합니다.

### 2. 기본 Service 통신

검증:

- 신규 curl Pod 생성
- CoreDNS metrics 접근
- Kubernetes API Service 접근
- metrics API Available True 확인

결과:

- CoreDNS metrics 접근 성공
- API Server `/healthz`는 `401 Unauthorized` 응답으로 네트워크 도달 확인
- `metrics.k8s.io` Available True

판정:

- `kubeProxyReplacement=true` 전환 후에도 기본 ClusterIP/DNS/API 경로는 유지됐습니다.

### 3. Gateway API north-south

검증:

- `GatewayClass/cilium` 생성 확인
- `Gateway` Accepted/Programmed True 확인
- `HTTPRoute` Accepted/ResolvedRefs True 확인
- NKS LoadBalancer IP 할당 확인
- 외부 LoadBalancer IP 요청
- Gateway Service 내부 경유 요청
- Gateway NodePort 요청
- Gateway hostNetwork 요청
- Cilium Ingress hostNetwork 요청

결과:

| 항목 | 결과 |
| --- | --- |
| Gateway 제어면 | Accepted/Programmed True |
| HTTPRoute 제어면 | Accepted/ResolvedRefs True |
| LoadBalancer | IP 할당 |
| 외부 LB HTTP 요청 | `Empty reply from server` |
| Gateway Service 내부 요청 | timeout |
| Gateway NodePort | timeout 또는 connection refused |
| Gateway hostNetwork | 일부 응답, 일부 upstream timeout |
| Cilium Ingress hostNetwork | 일부 응답, 일부 upstream timeout |

판정:

- 제어면은 정상처럼 보이지만 datapath가 운영 수준으로 완성되지 않았습니다.
- 보안그룹 또는 NKS LoadBalancer만의 문제로 보기 어렵습니다.

### 4. 일반 NodePort 대조군

검증:

- Cilium Gateway가 아닌 일반 backend Service를 NodePort로 노출
- worker node IP:NodePort로 요청

결과:

- 일반 NodePort 요청 성공

판정:

- NKS 보안그룹이 NodePort 대역을 전부 막아서 실패한 것이 아닙니다.
- Cilium Gateway/Ingress Envoy 경로에서 문제가 발생했습니다.

### 5. GAMMA east-west

검증:

- `HTTPRoute` parentRef를 `Service/web`으로 지정
- backendRef를 `web-v1`, `web-v2`로 지정
- `HTTPRoute` 상태 확인
- `CiliumEnvoyConfig`, Envoy listener/cluster 생성 확인
- 내부 Pod에서 `http://web/` 호출

결과:

| 항목 | 결과 |
| --- | --- |
| HTTPRoute 생성 | 성공 |
| HTTPRoute 상태 | Accepted/ResolvedRefs True |
| CiliumEnvoyConfig 생성 | 확인 |
| Envoy listener/cluster 생성 | 확인 |
| BPF LB map L7 proxy port | 확인 |
| 실제 `http://web/` 요청 | timeout |
| HTTPRoute 제거 후 일반 Service 요청 | 정상 복구 |

판정:

- GAMMA 제어면은 동작하지만 실제 Pod -> Service -> Envoy -> backend 경로가 실패합니다.
- “내부 Pod 간 Service Mesh가 된다”고 판정할 수 없습니다.

### 6. Cilium 공식 connectivity-check

검증:

- 공식 Cilium connectivity-check manifest를 임시 namespace에 적용
- ClusterIP, multi-node ClusterIP, headless, NodePort, CNP allow/deny 확인

결과:

| 항목 | 결과 |
| --- | --- |
| 일반 ClusterIP | 통과 |
| multi-node ClusterIP | 통과 |
| headless | 통과 |
| NodePort | 통과 |
| CNP allow/deny | 통과 |
| FQDN CNP 외부 DNS 항목 | timeout |

판정:

- 기본 Cilium chaining/KPR 전체가 망가진 것이 아닙니다.
- L7 proxy 또는 DNS/FQDN proxy 성격의 기능에서 제한이 드러납니다.

### 7. HTTP L7 CiliumNetworkPolicy 대조 검증

검증:

1. `cilium-l7-check` namespace에 `curl` Pod와 `nginx:1.27-alpine` backend 생성
2. `echo` Service와 backend Pod IP로 일반 HTTP 요청
3. HTTP L7 CiliumNetworkPolicy 적용
4. 같은 요청 재시도
5. L7 policy 삭제 후 같은 요청 재시도

결과:

| 조건 | 결과 |
| --- | --- |
| L7 policy 적용 전 `http://echo/` | nginx `200` |
| L7 policy 적용 전 `http://ClusterIP/` | nginx `200` |
| L7 policy 적용 전 `http://PodIP/` | nginx `200` |
| HTTP L7 policy 적용 후 `http://echo/` | timeout |
| HTTP L7 policy 적용 후 `http://PodIP/` | timeout |
| L7 policy 삭제 후 `http://echo/` | `200` |

판정:

- 실패 범위가 GAMMA만이 아니라 Cilium HTTP L7 proxy 전반임을 확인했습니다.
- 현재 chaining 구조에서 egress L7 policy redirect 이후 Envoy listener까지 연결이 전달되지 않는 것이 핵심입니다.

## 상세 실패 지점

2026-05-13 추가 검증으로 timeout 지점을 아래처럼 좁혔습니다.

### 테스트 구성

```text
namespace: cilium-l7-check
client: curlimages/curl:8.10.1, label app=curl
backend: nginx:1.27-alpine, label app=echo
Service: echo, ClusterIP 10.254.221.57, port 80
client Pod IP: 10.100.38.39
backend Pod IP: 10.100.38.38
node: ta-sgh-vxlan-sidecarless-cls-default-worker-node-3
Cilium agent: cilium-lcrvd
Cilium Envoy: cilium-envoy-zmqgh
```

### 정상 baseline

L7 policy 적용 전에는 모두 정상입니다.

| 요청 | 결과 |
| --- | --- |
| `curl http://echo/` | nginx `200` |
| `curl http://10.254.221.57:80/` | nginx `200` |
| `curl http://10.100.38.38:80/` | nginx `200` |

이 결과로 아래 항목은 실패 원인에서 제외했습니다.

- backend nginx 장애
- Service selector/EndpointSlice 오류
- 같은 노드 Pod 간 통신 오류
- ClusterIP Service 기본 경로 오류

### L7 policy 적용 후 Cilium 상태

적용한 정책:

```text
CiliumNetworkPolicy/allow-root-only
endpointSelector: app=curl
egress toEndpoints: app=echo
port: 80/TCP
http rule: GET /$
```

Cilium endpoint 상태:

| 항목 | 값 |
| --- | --- |
| client endpoint id | `1247` |
| backend endpoint id | `121` |
| client endpoint policy | `egress` |
| egress L4 rule | `app=echo`, `80/TCP` |
| allocated proxy port | `13147` |
| endpoint proxy statistics | requests `{}`, responses `{}` |

BPF policy map:

```text
Allow Egress app=echo 80/TCP PROXY PORT 13147
BYTES 444
PACKETS 6
```

Cilium status:

```text
Proxy Status: OK, 1 redirects active, Envoy: external
```

Envoy 상태:

```text
listener: cilium-http-egress:13147
address: 127.0.0.1:13147
```

Envoy metric:

```text
envoy_listener_connections_accepted_per_socket_event_count{envoy_listener_address="127.0.0.1_13147"} 0
envoy_http_downstream_rq_time_count{envoy_http_conn_manager_prefix="proxy"} 0
```

Cilium/Envoy 로그:

```text
Envoy: Upserting new listener listener=cilium-http-egress:13147
Adding new proxy port rules id=cilium-http-egress proxyPort=13147
lds: add/update listener 'cilium-http-egress:13147'
```

Cilium monitor:

```text
Policy verdict log:
local EP ID 1247, remote ID 32674, proto 6, egress,
action redirect, match L3-L4,
10.100.38.39:<port> -> 10.100.38.38:80 tcp SYN

-> proxy port 13147
identity 60959->unknown state new
10.100.38.39:<port> -> 10.100.38.38:80 tcp SYN
```

### 상세 판정

패킷 흐름은 아래 지점까지 확인됐습니다.

```text
curl Pod
  -> backend Pod 또는 Service:80
  -> Cilium endpoint egress policy match
  -> BPF policy map에서 proxy port 13147 확인
  -> Cilium monitor에 action redirect 기록
  -> 여기서 timeout
  -> Envoy listener 127.0.0.1:13147 accepted connection 0
```

따라서 실패 위치는 아래입니다.

```text
Cilium eBPF policy redirect
  -> local Envoy listener accept
```

즉, 정책 매칭과 redirect verdict는 발생하지만, redirect된 TCP SYN이 Envoy listener에 실제 연결로 수락되지 않습니다.

이로 인해 다음 단계는 실행되지 않습니다.

- Envoy HTTP request parsing
- L7 policy allow/deny 판단
- upstream backend 연결
- Envoy/Cilium proxy L7 metric 증가

### 상세 실패 사유

확정된 실패 사유:

> 현재 NKS Calico primary + Cilium generic-veth chaining 환경에서는 Cilium egress L7 policy redirect가 Envoy listener까지 연결을 전달하지 못해 HTTP 요청이 timeout됩니다.

유력한 기술 원인 범위:

- Calico가 만든 veth/TC datapath 위에 Cilium generic-veth chaining으로 붙은 상태에서 Cilium L7 proxy redirect 경로가 정상 동작하지 않음
- endpoint egress BPF policy map의 proxy port redirect와 host-local Envoy listener 사이의 전달 경로 문제
- Service/Gateway/GAMMA 문제가 아니라 Cilium L7 proxy 공통 경로 문제

이번 검증으로 배제한 항목:

| 항목 | 배제 근거 |
| --- | --- |
| backend 애플리케이션 문제 | policy 적용 전/삭제 후 nginx `200` |
| Service/Endpoint 문제 | ClusterIP와 PodIP 모두 baseline `200` |
| DNS 문제 | 직접 ClusterIP/PodIP 요청도 동일 증상 |
| CNP validation 문제 | CNP `Valid=True`, endpoint egress policy 반영 |
| Envoy listener 미생성 | `cilium-http-egress:13147` listener 생성 확인 |
| Envoy upstream/backend 문제 | Envoy listener accepted connection 자체가 0 |
| 보안그룹/NodePort 문제 | 같은 노드 PodIP 직접 요청도 L7 policy 적용 시 timeout |

남은 미확정 저수준 원인:

- Cilium BPF redirect helper 이후 host stack 전달 세부 동작
- Calico veth/TC hook과 Cilium L7 proxy redirect의 상호작용
- Cilium external Envoy + `generic-veth` chaining 조합의 커널 datapath 세부 제약

다만 운영 판단에는 위 미확정 항목까지 추가로 규명하지 않아도 충분합니다.
동일 증상이 GAMMA/Gateway/HTTP L7 Policy에서 반복되고, L7 policy 제거 시 즉시 복구되기 때문에 현재 구성은 Cilium Service Mesh 운영 조건을 만족하지 못합니다.

## 실패 원인 분석

### 원인이 아닌 것

아래 항목은 단독 원인으로 보기 어렵습니다.

- Cilium 전체 장애
- Calico 전체 장애
- CoreDNS 장애
- metrics-server 장애
- NKS 보안그룹의 NodePort 전체 차단
- Gateway API CRD 미설치
- Gateway/HTTPRoute 제어면 실패
- kube-proxy DaemonSet 잔존 충돌

### 핵심 원인

현재 구조는 Calico가 primary CNI입니다.
Pod veth, Pod IPAM, VXLAN datapath는 Calico가 담당합니다.
Cilium은 `generic-veth` chaining으로 Calico가 만든 veth에 eBPF 프로그램을 붙입니다.

이 구조에서는 L3/L4 관측과 일부 정책은 가능하지만, Cilium Service Mesh에 필요한 아래 경로가 안정적으로 동작하지 않았습니다.

```text
client Pod
  -> ClusterIP Service
  -> Cilium eBPF service handling
  -> Cilium eBPF policy redirect
  -> Cilium Envoy L7 listener
  -> HTTPRoute/L7 policy 처리
  -> backend Pod
```

공식 Cilium chaining 문서도 chaining 환경에서 Layer 7 Policy 같은 고급 기능이 제한될 수 있다고 명시합니다.
이번 PoC 결과는 해당 제한이 실제 NKS Calico chaining 환경에서 Cilium Service Mesh 실패로 나타난 사례입니다.

## kube-proxy에 대한 정확한 해석

기존 NKS Calico 구조는 보통 아래처럼 이해할 수 있습니다.

```text
Calico: Pod 네트워크
kube-proxy: Kubernetes Service 처리
```

Cilium Service Mesh에서는 아래처럼 바뀌어야 합니다.

```text
Cilium eBPF: kube-proxy replacement
Cilium Envoy: L7 proxy
```

따라서 “Envoy가 kube-proxy를 대체한다”는 표현은 틀립니다.
정확히는 “Cilium eBPF가 kube-proxy를 대체하고, Envoy는 Cilium eBPF가 넘긴 L7 트래픽을 처리한다”입니다.

이번 검증에서는 `kubeProxyReplacement=true` 자체는 활성화됐고 일반 Service 통신도 유지됐습니다.
그러나 Envoy L7 redirect 경로가 timeout으로 실패했습니다.

## 왜 Cilium primary가 필요한가

Cilium Service Mesh를 운영 수준으로 쓰려면 Cilium이 Service 처리와 L7 redirect datapath를 안정적으로 소유해야 합니다.

운영 가능한 기준 구조는 아래에 가깝습니다.

```text
Pod datapath: Cilium primary CNI
Service datapath: Cilium kube-proxy replacement
L7 datapath: Cilium Envoy
```

반대로 현재 구조는 아래입니다.

```text
Pod datapath: Calico primary CNI
Service datapath: Cilium KPR 일부 활성화
L7 datapath: Cilium Envoy redirect 실패
```

따라서 Calico가 “설치되어 있기 때문”이 아니라, **Calico가 primary datapath를 계속 소유한 상태에서 Cilium chaining만으로 L7 Service Mesh datapath를 완성하지 못하기 때문**입니다.

## NKS full replacement 경로

Cilium Service Mesh의 정석 경로는 Cilium primary CNI 또는 kube-proxy-free Cilium datapath입니다.
하지만 이 환경에서는 이전 Track A에서 Calico를 Cilium으로 full replacement하는 시도도 실패로 정리됐습니다.

따라서 현재 NKS에서 가능한 결론은 아래입니다.

| 경로 | 판정 |
| --- | --- |
| Calico 유지 + Cilium chaining + Hubble | 가능 |
| Calico 유지 + Cilium chaining + Cilium Service Mesh | 불가 |
| Calico 제거/대체 + Cilium primary | 이전 검증 실패 |
| Cilium Gateway/Ingress north-south | 불가 |
| Cilium GAMMA east-west | 불가 |
| Cilium HTTP L7 Policy | 불가 |

## 운영 권장안

현재 NKS 기본 Calico 클러스터에서는 아래 구성이 현실적인 운영 후보입니다.

```text
NKS 기본 Calico VXLAN
  + kube-proxy Service 처리
  + Cilium generic-veth chaining
  + Hubble/Grafana L3/L4 flow 관측
```

고객 요구가 Service Mesh라면 아래 중 하나로 범위를 조정해야 합니다.

| 요구 | 권장 방향 |
| --- | --- |
| 네트워크 flow 관측 | 현재 Cilium chaining + Hubble 유지 |
| HTTP/L7 metrics 일부 | 별도 Hubble/L7 관측을 켠 경우에도 Envoy/L7 proxy 경유 트래픽에 한정됨을 명시 |
| sidecar-less mesh | Istio Ambient 별도 PoC |
| mTLS, L7 authorization, 서비스 그래프 | Istio Ambient 별도 PoC |
| Cilium Service Mesh 필수 | Cilium primary CNI가 가능한 별도 클러스터에서 PoC |

## 문서 반영 기준

- 실행 가이드는 현재 NKS chaining에 억지로 Service Mesh를 구성하는 절차로 두지 않습니다.
- 별도 가이드에는 Cilium primary 설치 시 Service Mesh 구성, Hubble 관측 계층, Envoy/Cilium proxy 검증 절차를 정리합니다.
- 현재 NKS에서 실패한 증거와 판정은 이 보고서에 유지합니다.

## 참고 출처

- Cilium CNI Chaining: https://docs.cilium.io/en/stable/installation/cni-chaining/
- Cilium Calico Chaining: https://docs.cilium.io/en/stable/installation/cni-chaining-calico/
- Cilium Gateway API: https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/
- Cilium GAMMA: https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gamma/
- Cilium L7 Traffic Management: https://docs.cilium.io/en/stable/network/servicemesh/l7-traffic-management/
- Cilium kube-proxy replacement: https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/
