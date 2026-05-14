# 공식 근거 요약

- 목적: 현재 PoC 범위에 직접 영향을 주는 공식 문서 근거를 빠르게 확인합니다.
- 상태: draft
- 마지막 갱신: 2026-05-14

## NHN Cloud NKS

- 출처:
  - https://docs.nhncloud.com/en/Container/NKS/en/release-notes/
  - https://docs.nhncloud.com/ko/Container/NKS/ko/release-notes/
  - https://docs.nhncloud.com/en/Container/NKS/en/user-guide/
- 확인 날짜: 2026-04-22
- 핵심 사항
  - 2025-05-27 릴리스 노트에는 `The feature to change cluster CNI is no longer supported.`가 명시되어 있습니다.
  - 2025-11-25 릴리스 노트에는 Kubernetes `v1.33.4` 지원과 Calico CNI `v3.30.2-nks1` 애드온 추가가 보입니다.
  - 2025-12-23 한국어 릴리스 노트에는 플랫폼 버전 `1.202511.1` 추가가 보입니다.
  - 사용자 가이드에는 NKS worker node custom image 변환용 Ubuntu `Ubuntu Server 24.04.3 LTS (2025.11.18)`와 애플리케이션 버전 `1.9`가 보입니다.
  - 사용자 가이드에는 Calico `v3.24.1` eBPF 모드 클러스터가 Rocky 9.5 이상 또는 Ubuntu 24.04 이상 이미지로 node group을 생성할 수 없으며, 이 이미지를 쓰려면 Calico `v3.28.2` 이상 업데이트가 필요하다는 주의가 있습니다.
- 판단
  - NKS에서 Calico에서 Cilium으로의 전환은 공식 지원 시나리오로 보기 어렵습니다.
  - 따라서 이번 작업은 기능 검증 중심의 `unsupported PoC`로 관리해야 합니다.
  - Ubuntu 24.04를 사용할 경우 실제 콘솔에서 선택 가능한 worker image명과 Calico 애드온 버전을 생성 직전에 확인해야 합니다.

## Cilium Kubernetes 요구사항

- 출처: https://docs.cilium.io/en/stable/network/kubernetes/requirements.html
- 확인 날짜: 2026-04-22
- 핵심 사항
  - Cilium stable은 Kubernetes `1.31`, `1.32`, `1.33`, `1.34`를 지원한다고 명시합니다.
- 판단
  - 최신 NKS Kubernetes 버전과 기술적 호환 가능성은 있습니다.
  - 단, NKS 관리형 제약은 별도로 남습니다.

## Cilium 단일 CNI와 신규 노드 주의사항

- 출처: https://docs.cilium.io/en/stable/installation/taints/
- 확인 날짜: 2026-04-22
- 핵심 사항
  - Cilium이 지원되지 않는 단일 CNI 환경에서는 cloud provider가 재부팅, 업데이트, 유지보수 중 CNI 설정을 되돌릴 수 있습니다.
  - Cilium이 뜨기 전에 Pod가 먼저 시작되면 기존 CNI로 IP를 받을 수 있습니다.
  - Cilium은 `node.cilium.io/agent-not-ready` taint를 이용해 Cilium 준비 전 Pod 실행을 막는 방식을 안내합니다.
  - Cilium Helm에는 `agentNotReadyTaintKey` 값이 있으며, Cilium agent toleration은 기본적으로 taint가 있는 노드에도 스케줄될 수 있도록 구성됩니다.
  - Cluster Autoscaler가 taint를 무시해야 하는 경우 `ignore-taint.cluster-autoscaler.kubernetes.io/`로 시작하는 key 사용을 안내합니다.
  - Cilium 문서는 지원되지 않는 단일 CNI 환경에서 이미 스케줄된 Pod가 Cilium보다 먼저 실행되는 문제를 줄이기 위해 `NoExecute` effect를 권장합니다.
- 판단
  - NKS에서 Cilium full replacement를 검증하려면 신규 node scale-out 시에도 Cilium이 CNI 소유권을 가져가는지 별도 검증해야 합니다.
  - node group 수준 taint 또는 동등 자동화 없이는 scale-out 자동 안정성을 보장하지 않습니다.
  - NKS node group taint key와 Cilium `agentNotReadyTaintKey`가 일치해야 자동화 흐름이 성립합니다.
  - 기존 노드는 최초 1회 node-by-node migration이 필요하며, 신규 노드는 node group taint와 Cilium taint 제거 동작이 함께 확인되어야 자동화된 것으로 판단할 수 있습니다.

## Hubble 요구사항

- 출처: https://docs.cilium.io/en/stable/observability/hubble/setup/
- 확인 날짜: 2026-04-22
- 핵심 사항
  - Hubble Relay를 위해 모든 Cilium 노드에서 `TCP 4244`가 필요합니다.
- 판단
  - 보안그룹 또는 노드 간 통신 경로 검토가 선행되어야 합니다.

## CNI 보안그룹 포트

- 출처:
  - https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements
  - https://docs.tigera.io/calico/latest/reference/felix/configuration
  - https://docs.cilium.io/en/stable/operations/system_requirements/
- 확인 날짜: 2026-04-22
- 핵심 사항
  - Calico VXLAN은 `UDP 4789`를 사용합니다.
  - Cilium VXLAN tunnel mode는 기본적으로 `UDP 8472`를 사용합니다.
  - Cilium health는 노드 간 `TCP 4240`과 ICMP를 사용할 수 있습니다.
  - Hubble server는 `TCP 4244`를 사용합니다.
- 판단
  - 사용자가 제공한 NKS 기본 worker 보안그룹에는 worker self 원격 `TCP 1-65535`, `UDP 1-65535` 수신 허용이 있어 Cilium 기본 포트가 포함됩니다.
  - 따라서 현재 보안그룹을 유지한다면 Cilium VXLAN/Hubble 용도의 추가 규칙은 필요하지 않은 것으로 봅니다.
  - 실패로 종결한 full replacement 실험은 Cilium VXLAN 기본 `8472`를 사용했습니다.
  - NKS 관리형 Calico 관련 규칙은 PoC 중 삭제하지 않습니다.

## Hubble 모니터링 방식

- 출처:
  - https://docs.cilium.io/en/stable/observability/hubble/hubble-ui/
  - https://docs.cilium.io/en/stable/observability/grafana/
  - https://docs.cilium.io/en/stable/observability/metrics/
  - https://docs.cilium.io/en/stable/observability/hubble/configuration/export/
- 확인 날짜: 2026-04-22
- 핵심 사항
  - Hubble UI 공식 접근 방식은 `cilium hubble ui` 기반 port-forward입니다.
  - Cilium 예제 Prometheus/Grafana는 Cilium과 Hubble metrics를 자동 scrape하는 구성을 제공합니다.
  - Hubble metrics는 `hubble.metrics.enabled`로 활성화하며 Prometheus/Grafana에서 `hubble_` 계열 지표로 조회합니다.
  - Cilium/Hubble/Cilium Operator metrics 활성화 시 기본 scrape port는 각각 `9962`, `9965`, `9963`입니다.
  - Hubble exporter는 flow를 파일 또는 stdout으로 내보내 로그 수집기가 소비할 수 있습니다.
- 판단
  - PoC 기본 접근은 Hubble UI/CLI port-forward가 가장 안전합니다.
  - 운영형 상시 모니터링은 Hubble UI보다 Prometheus/Grafana metrics가 적합합니다.
  - 개별 flow 장기 검색과 감사는 Hubble exporter와 로그 플랫폼 연계를 별도 검토해야 합니다.
  - Hubble UI를 웹으로 공유할 경우 공인 무방비 노출이 아니라 내부 Ingress, TLS, 접근 제한을 전제로 해야 합니다.

## Cilium Ingress 특성

- 출처: https://docs.cilium.io/en/stable/network/servicemesh/ingress.html
- 확인 날짜: 2026-04-22
- 핵심 사항
  - Cilium Ingress는 기본적으로 LoadBalancer 또는 NodePort 기반으로 노출됩니다.
  - source IP visibility는 프록시 홉 수와 `X-Forwarded-For` 동작에 영향을 받습니다.
- 판단
  - 현재 PoC에서는 NKS LoadBalancer 기반 구성이 1차 후보입니다.
  - 실제 프록시 체인 연동 전에는 샘플 워크로드 기준으로 먼저 검증합니다.

## Cilium Gateway API 특성

- 출처: https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/
- 확인 날짜: 2026-04-22
- 핵심 사항
  - Cilium stable 1.19.x 문서 기준 Gateway API `v1.4.1`을 지원합니다.
  - `GatewayClass`, `Gateway`, `HTTPRoute`, `GRPCRoute`, `TLSRoute`, `ReferenceGrant`를 다룹니다.
  - 사용 전 Gateway API CRD를 사전 설치해야 합니다.
  - 기본 노출 방식은 `LoadBalancer`이고, host network 모드도 지원합니다.
- 판단
  - 장기적으로는 Ingress보다 더 구조적이고 확장성이 높습니다.
  - 현재 PoC에서는 Ingress보다 진입 복잡도가 높지만, 후속 표준 후보로 검토 가치가 큽니다.

## Cilium Calico Chaining 특성

- 출처: https://docs.cilium.io/en/stable/installation/cni-chaining-calico/
- 확인 날짜: 2026-04-22
- 핵심 사항
  - Calico 위에 Cilium chaining 구성이 가능합니다.
  - 일부 고급 기능은 제한될 수 있습니다.
  - 기존 Pod는 재시작해야 새 chaining 구성이 적용됩니다.
- 판단
  - Cilium chaining + Hubble은 NKS 기본 Calico를 유지하면서 네트워크 flow 관측을 추가하는 낮은 리스크 경로입니다.
  - 다만 full replacement 또는 Cilium Service Mesh 목표와는 성격이 다릅니다.
