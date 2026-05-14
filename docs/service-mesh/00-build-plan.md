# Cilium Service Mesh 검증 계획과 판정

- 목적: NKS Calico 환경에서 Cilium Service Mesh를 검토한 범위, 검증 순서, 최종 판정을 정리합니다.
- 상태: concluded
- 마지막 갱신: 2026-05-14

## 최종 판정

현재 NKS `Calico primary CNI + Cilium generic-veth chaining` 구조에서는 Cilium Service Mesh를 운영 수준으로 구성하지 않습니다.

판정:

| 항목 | 결과 |
| --- | --- |
| Calico 유지 + Cilium chaining + Hubble | 가능 |
| Cilium Gateway API north-south | 실패 |
| Cilium Ingress north-south | 실패 |
| Cilium GAMMA east-west | 실패 |
| Cilium HTTP L7 Policy | 실패 |
| Cilium full replacement | 이전 Track A에서 실패 |

상세 결과는 `docs/service-mesh/05-build-result-report.md`를 기준으로 합니다.

## 검증 목적

고객이 Service Mesh 도입을 요구할 경우를 대비해 아래 질문을 확인했습니다.

1. Calico를 primary CNI로 유지한 상태에서 Cilium chaining으로 Service Mesh가 가능한가
2. Cilium Gateway API로 외부 north-south 트래픽을 처리할 수 있는가
3. Cilium GAMMA로 내부 east-west Service 트래픽을 L7 라우팅할 수 있는가
4. Cilium Envoy/L7 proxy 경로가 실제 datapath로 성립하는가
5. 실패한다면 어디까지 가능한지 운영 범위를 정할 수 있는가

## 검증 순서

| 단계 | 검증 내용 | 결과 |
| --- | --- | --- |
| 1 | Gateway API CRD 설치 | 성공 |
| 2 | `kubeProxyReplacement=true`, `gatewayAPI.enabled=true`, `l7Proxy=true` 적용 | 성공 |
| 3 | 기본 ClusterIP/DNS/API/metrics 통신 | 성공 |
| 4 | Gateway API LoadBalancer | 제어면 성공, HTTP 실패 |
| 5 | Gateway NodePort | 실패 |
| 6 | Gateway hostNetwork | 불안정 |
| 7 | Cilium Ingress hostNetwork | 불안정 |
| 8 | GAMMA Service parentRef HTTPRoute | 제어면 성공, 실제 요청 실패 |
| 9 | 공식 connectivity-check | 기본 L3/L4 대부분 성공 |
| 10 | HTTP L7 CiliumNetworkPolicy | 적용 시 timeout, 제거 시 복구 |

## 실패 기준

Cilium Service Mesh 실패 판정은 단순히 Gateway/HTTPRoute 리소스가 생성되는지가 아니라 실제 datapath 기준으로 판단했습니다.

실패 기준:

- Gateway/HTTPRoute가 Accepted/Programmed True여도 실제 HTTP 요청 실패
- GAMMA HTTPRoute가 Accepted/ResolvedRefs True여도 parent Service 요청 timeout
- HTTP L7 CiliumNetworkPolicy 적용 시 일반 Service 요청 timeout
- L7 policy 제거 후 같은 요청이 복구됨

## 운영 범위

현재 NKS에서 유지 가능한 운영 범위:

```text
NKS 기본 Calico VXLAN
  + kube-proxy Service 처리
  + Cilium generic-veth chaining
  + Hubble/Grafana L3/L4 flow 관측
```

현재 NKS에서 운영 범위에서 제외하는 항목:

```text
Cilium Gateway API
Cilium Ingress
Cilium GAMMA
Cilium HTTP L7 Policy
Cilium Service Mesh
```

## 후속 가이드

Cilium Service Mesh 자체의 표준 구축 절차는 `docs/service-mesh/01-cilium-service-mesh-runbook.md`에 분리합니다.
해당 가이드는 Cilium primary CNI와 kube-proxy replacement가 가능한 클러스터를 대상으로 하며, 현재 NKS Calico chaining 클러스터에 직접 적용하지 않습니다.

## 참고 문서

- `docs/service-mesh/01-cilium-service-mesh-runbook.md`
- `docs/service-mesh/03-cilium-service-mesh-concepts.md`
- `docs/service-mesh/04-operational-weight-comparison.md`
- `docs/service-mesh/05-build-result-report.md`
