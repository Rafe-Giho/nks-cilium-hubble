# 공식 근거 요약

- 목적: 현재 PoC 범위에 직접 영향을 주는 공식 문서 근거를 빠르게 확인합니다.
- 상태: draft
- 마지막 갱신: 2026-04-16

## NHN Cloud NKS

- 출처: https://docs.nhncloud.com/en/Container/NKS/en/release-notes/
- 확인 날짜: 2026-04-16
- 핵심 사항
  - 2025-05-27 릴리스 노트에는 `The feature to change cluster CNI is no longer supported.`가 명시되어 있습니다.
  - 2025-11-25 릴리스 노트에는 Kubernetes `v1.33.4` 지원이 보입니다.
- 판단
  - NKS에서 Calico에서 Cilium으로의 전환은 공식 지원 시나리오로 보기 어렵습니다.
  - 따라서 이번 작업은 기능 검증 중심의 `unsupported PoC`로 관리해야 합니다.

## Cilium Kubernetes 요구사항

- 출처: https://docs.cilium.io/en/stable/network/kubernetes/requirements.html
- 확인 날짜: 2026-04-16
- 핵심 사항
  - Cilium stable은 Kubernetes `1.31`, `1.32`, `1.33`, `1.34`를 지원한다고 명시합니다.
- 판단
  - 최신 NKS Kubernetes 버전과 기술적 호환 가능성은 있습니다.
  - 단, NKS 관리형 제약은 별도로 남습니다.

## Hubble 요구사항

- 출처: https://docs.cilium.io/en/stable/observability/hubble/setup/
- 확인 날짜: 2026-04-16
- 핵심 사항
  - Hubble Relay를 위해 모든 Cilium 노드에서 `TCP 4244`가 필요합니다.
- 판단
  - 보안그룹 또는 노드 간 통신 경로 검토가 선행되어야 합니다.

## Cilium Ingress 특성

- 출처: https://docs.cilium.io/en/stable/network/servicemesh/ingress.html
- 확인 날짜: 2026-04-16
- 핵심 사항
  - Cilium Ingress는 기본적으로 LoadBalancer 또는 NodePort 기반으로 노출됩니다.
  - source IP visibility는 프록시 홉 수와 `X-Forwarded-For` 동작에 영향을 받습니다.
- 판단
  - 현재 PoC에서는 NKS LoadBalancer 기반 구성이 1차 후보입니다.
  - 실제 프록시 체인 연동 전에는 샘플 워크로드 기준으로 먼저 검증합니다.
