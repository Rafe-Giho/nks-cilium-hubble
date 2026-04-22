# 공식 근거 요약

- 목적: 현재 PoC 범위에 직접 영향을 주는 공식 문서 근거를 빠르게 확인합니다.
- 상태: draft
- 마지막 갱신: 2026-04-22

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

## Hubble 요구사항

- 출처: https://docs.cilium.io/en/stable/observability/hubble/setup/
- 확인 날짜: 2026-04-22
- 핵심 사항
  - Hubble Relay를 위해 모든 Cilium 노드에서 `TCP 4244`가 필요합니다.
- 판단
  - 보안그룹 또는 노드 간 통신 경로 검토가 선행되어야 합니다.

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
  - Cilium stable 1.19.3 문서 기준 Gateway API `v1.4.1`을 지원합니다.
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
  - sidecar-less 네트워크 가시성만 빨리 보려면 낮은 리스크 대안입니다.
  - 다만 full replacement 목표와는 성격이 다릅니다.

## Pixie 특성

- 출처: https://docs.px.dev/about-pixie/what-is-pixie/
- 확인 날짜: 2026-04-22
- 핵심 사항
  - Pixie는 eBPF 기반 Kubernetes 관측 도구입니다.
  - 네트워크, 서비스 맵, 요청 추적, 프로파일링 등 폭넓은 관측을 제공합니다.
  - Hosted 방식은 외부 Pixie Cloud와의 통신을 요구할 수 있습니다.
- 판단
  - sidecar-less 관측 목적에는 매우 잘 맞습니다.
  - 하지만 CNI/Ingress replacement 성격은 아닙니다.

## Grafana Beyla 특성

- 출처: https://grafana.com/docs/beyla/latest/
- 확인 날짜: 2026-04-22
- 핵심 사항
  - Beyla는 eBPF 기반 auto-instrumentation 도구입니다.
  - RED metrics와 basic traces 수집에 적합합니다.
  - Kubernetes DaemonSet, Helm 배포를 지원합니다.
  - Cilium과 함께 쓸 때 TCX 또는 TC priority 설정 호환성을 봐야 합니다.
- 판단
  - sidecar-less 앱 관측 후보로 적합합니다.
  - Hubble와 같은 네트워크 중심 UX를 대체하는 것은 아닙니다.
