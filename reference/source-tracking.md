# 출처 추적

- 목적: 공식 문서와 확인 시점을 한곳에서 추적합니다.
- 상태: draft
- 마지막 갱신: 2026-04-16

| ID | 영역 | 출처 | 확인 날짜 | 버전/문서 시점 | 핵심 요약 | 영향 |
| --- | --- | --- | --- | --- | --- | --- |
| R-001 | NKS CNI 지원 | https://docs.nhncloud.com/en/Container/NKS/en/release-notes/ | 2026-04-16 | 2025-05-27 / 2025-11-25 release notes | NKS 릴리스 노트 기준 cluster CNI change 기능은 더 이상 지원되지 않음. 최신 릴리스에는 Kubernetes v1.33.4 지원이 보임. | 비지원 PoC 전제와 버전 기준선 설정 |
| R-002 | Cilium 지원 매트릭스 | https://docs.cilium.io/en/stable/network/kubernetes/requirements.html | 2026-04-16 | stable docs | Cilium stable은 Kubernetes 1.31~1.34 지원을 명시 | NKS 최신 버전과의 기술적 호환성 검토 |
| R-003 | Hubble 요구사항 | https://docs.cilium.io/en/stable/observability/hubble/setup/ | 2026-04-16 | stable docs | Hubble Relay 동작을 위해 모든 Cilium 노드에서 TCP 4244 오픈 필요 | 보안그룹/네트워크 경로 검토 필요 |
| R-004 | Ingress 동작 특성 | https://docs.cilium.io/en/stable/network/servicemesh/ingress.html | 2026-04-16 | stable docs | Cilium Ingress는 기본적으로 LoadBalancer/NodePort 기반으로 노출되고 source IP visibility와 trusted hop 동작이 중요 | Ingress 설계와 후속 앱 연동 검토 기준 |
