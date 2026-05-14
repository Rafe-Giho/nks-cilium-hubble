# Cilium Service Mesh 검증 체크리스트

- 목적: Cilium Service Mesh PoC 적용 전후의 성공 기준과 중단 기준을 추적합니다.
- 상태: draft
- 마지막 갱신: 2026-05-14

## 적용 전

실제 실행 결과와 운영 가능/불가 판정은 `docs/service-mesh/05-build-result-report.md`에 기록합니다.
이 문서는 검증 항목과 성공 기준만 정의합니다.

| 항목 | 성공 기준 |
| --- | --- |
| 노드 상태 | worker 노드 모두 `Ready` |
| Calico 상태 | `calico-node` Ready |
| Cilium 상태 | `cilium status` OK |
| Cilium Envoy | DaemonSet Ready |
| Hubble Relay/UI | Ready. 관측 계층 성공 기준이며, datapath 성공 기준은 Envoy/Gateway/GAMMA 요청으로 별도 판단 |
| metrics API | `v1beta1.metrics.k8s.io` Available True |
| Gateway API CRD | 최초 구축 전이면 미설치, 재실행이면 버전/소유자 확인 |
| `kubeProxyReplacement` | 최초 구축 전이면 `false`, 4번 이후 재검증이면 `true` |
| LoadBalancer 사용 가능성 | 샘플 Service 또는 Gateway Service로 주소 할당과 실제 HTTP 응답을 모두 확인 |

## Gateway API CRD

| 항목 | 성공 기준 |
| --- | --- |
| 설치 방식 | 공식 `standard-install.yaml` release bundle 또는 저장소 wrapper 사용 |
| server-side apply | `kubectl apply --server-side` 사용 |
| GatewayClass CRD | 존재 |
| Gateway CRD | 존재 |
| HTTPRoute CRD | 존재 |
| GRPCRoute CRD | 존재 |
| ReferenceGrant CRD | 존재 |
| Experimental CRD | TLSRoute/TCPRoute/UDPRoute가 필요할 때만 적용 |

## Helm 변경 검증

| 항목 | 성공 기준 |
| --- | --- |
| dry-run 성공 | Helm YAML 렌더링 성공 |
| chaining 설정 유지 | `generic-veth`, `/etc/cni/net.d/calico`, `exclusive=false` 유지 |
| Gateway API 활성화 | `gatewayAPI.enabled=true` |
| kube-proxy replacement | `kubeProxyReplacement=true` |
| Hubble 활성화 | Relay/UI/metrics/exporter 설정 렌더링 확인 |

## 적용 후 기본 상태

| 항목 | 성공 기준 |
| --- | --- |
| Cilium agent | 모든 Pod Ready |
| Cilium operator | Ready |
| Cilium Envoy | 모든 Pod Ready |
| Hubble Relay/UI | Ready |
| `cilium status --wait` | OK |
| CoreDNS | Ready |
| metrics API | Available True |
| 신규 Pod DNS | `kube-dns` 조회 성공 |
| 신규 Pod Service 통신 | Kubernetes/CoreDNS Service 접근 성공 |
| `GatewayClass/cilium` | 존재 |

## Gateway 샘플

| 항목 | 성공 기준 |
| --- | --- |
| GatewayClass | `cilium` 존재 |
| Gateway | Accepted=True, Programmed=True |
| HTTPRoute | Accepted=True, ResolvedRefs=True |
| LoadBalancer | 외부 주소 또는 FQDN 할당 |
| HTTP 요청 | `mesh-demo.example` 요청 성공 |
| Envoy/Cilium proxy 상태 | Gateway 요청 후 proxy metric 또는 로그에서 요청 처리 흔적 확인 |
| Hubble 관측 | Gateway 요청 후 flow 또는 HTTP L7 metric 확인 |

추가 대안 검증:

| 항목 | 성공 기준 |
| --- | --- |
| 일반 NodePort | node IP:NodePort 요청 성공 |
| Gateway NodePort | node IP:Gateway NodePort 요청 성공 |
| Gateway hostNetwork | node IP:8080 요청 성공 |
| Cilium Ingress hostNetwork | node IP:8081 요청 성공 |

## GAMMA 샘플

| 항목 | 성공 기준 |
| --- | --- |
| Service parentRef HTTPRoute | 생성 성공 |
| HTTPRoute 상태 | Accepted=True, ResolvedRefs=True |
| 내부 Pod 요청 | `http://web/` 성공 |
| Envoy/Cilium proxy 상태 | 요청 후 proxy metric 또는 로그에서 요청 처리 흔적 확인 |
| Hubble 관측 | Service/GAMMA 요청 후 flow 또는 HTTP L7 metric 확인 |

## 중단 기준

- Cilium agent 또는 operator가 10분 안에 복구되지 않음
- CoreDNS 또는 metrics API 비정상
- 신규 Pod DNS/Service 통신 실패
- Gateway API 적용 후 기존 Calico 네트워크에 영향 발생
- LoadBalancer가 장시간 Pending이고 hostNetwork 노출 승인이 없음

## 결과 기록 위치

- 단계별 실행 로그: `artifacts/service-mesh/`
- 현재 NKS PoC 결과: `docs/service-mesh/05-build-result-report.md`
- 주요 판단: `docs/decision-log.md`
