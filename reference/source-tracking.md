# 출처 추적

- 목적: 공식 문서와 확인 시점을 한곳에서 추적합니다.
- 상태: draft
- 마지막 갱신: 2026-05-06

| ID | 영역 | 출처 | 확인 날짜 | 버전/문서 시점 | 핵심 요약 | 영향 |
| --- | --- | --- | --- | --- | --- | --- |
| R-001 | NKS CNI 지원 | https://docs.nhncloud.com/en/Container/NKS/en/release-notes/ | 2026-04-22 | 2025-05-27 / 2025-11-25 release notes | NKS 릴리스 노트 기준 cluster CNI change 기능은 더 이상 지원되지 않음. Kubernetes v1.33.4와 Calico CNI v3.30.2-nks1 지원이 보임. | 비지원 PoC 전제와 버전 기준선 설정 |
| R-002 | Cilium 지원 매트릭스 | https://docs.cilium.io/en/stable/network/kubernetes/requirements.html | 2026-04-22 | stable docs | Cilium stable은 Kubernetes 1.31~1.34 지원을 명시 | NKS 최신 버전과의 기술적 호환성 검토 |
| R-003 | Hubble 요구사항 | https://docs.cilium.io/en/stable/observability/hubble/setup/ | 2026-04-22 | stable docs | Hubble Relay 동작을 위해 모든 Cilium 노드에서 TCP 4244 오픈 필요 | 보안그룹/네트워크 경로 검토 필요 |
| R-004 | Ingress 동작 특성 | https://docs.cilium.io/en/stable/network/servicemesh/ingress.html | 2026-04-22 | stable docs | Cilium Ingress는 기본적으로 LoadBalancer/NodePort 기반으로 노출되고 source IP visibility와 trusted hop 동작이 중요 | Ingress 설계와 후속 앱 연동 검토 기준 |
| R-005 | NKS 플랫폼/이미지 | https://docs.nhncloud.com/ko/Container/NKS/ko/release-notes/ | 2026-04-22 | 2025-12-23 release notes | 플랫폼 버전 1.202511.1 추가가 보임 | 클러스터/노드그룹 생성 시 플랫폼 버전 확인 필요 |
| R-006 | Ubuntu 24.04 이미지 제약 | https://docs.nhncloud.com/en/Container/NKS/en/user-guide/ | 2026-04-22 | user guide | Ubuntu Server 24.04.3 LTS (2025.11.18)와 애플리케이션 1.9가 worker image 변환 표에 보이며, Calico v3.24.1 eBPF 모드에서는 Ubuntu 24.04+ node group 생성 제약이 있음 | 실제 콘솔 선택 이미지명과 Calico 애드온 버전 확인 필요 |
| R-007 | Gateway API 지원 | https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/ | 2026-04-22 | stable docs | Cilium stable 1.19.3은 Gateway API v1.4.1 지원, Gateway API CRD 사전 설치 필요 | Ingress 대비 대안 설계와 장기 표준 검토 기준 |
| R-008 | Calico chaining | https://docs.cilium.io/en/stable/installation/cni-chaining-calico/ | 2026-04-22 | stable docs | Calico 위에 Cilium chaining 가능, 일부 기능 제한, 기존 Pod 재시작 필요 | 낮은 리스크 sidecar-less 관측 대안 검토 기준 |
| R-009 | Pixie 요구사항/설치 | https://docs.px.dev/installing-pixie/requirements/ | 2026-04-22 | current docs | Pixie는 K8s 1.21+, Linux kernel 4.14+, privileged access, cloud 443 통신 등을 요구 | Cilium 외 sidecar-less 관측 후보 검토 기준 |
| R-010 | Grafana Beyla 설치/호환성 | https://grafana.com/docs/beyla/latest/setup/kubernetes-helm/ | 2026-04-22 | latest docs | Beyla는 DaemonSet/Helm 설치 가능, Cilium과 TCX/priority 호환성 확인 필요 | 앱 관측 중심 sidecar-less 후보 검토 기준 |
| R-011 | Hubble UI/CLI/Grafana | https://docs.cilium.io/en/stable/observability/hubble/hubble-ui/ ; https://docs.cilium.io/en/stable/observability/hubble/hubble-cli/ ; https://docs.cilium.io/en/stable/observability/grafana/ ; https://docs.cilium.io/en/stable/observability/metrics/ | 2026-04-22 | stable docs | Hubble metrics는 Prometheus/Grafana로 시계열 모니터링 가능하며, Hubble UI는 port-forward 또는 Ingress 기반 접근이 가능 | PoC와 운영형 네트워크 플로우 모니터링 가이드 기준 |
| R-012 | Hubble exporter | https://docs.cilium.io/en/stable/observability/hubble/configuration/export/ | 2026-04-22 | stable docs | Hubble exporter는 flow를 파일 또는 stdout으로 내보내 로그 수집기가 소비할 수 있음 | 장기 검색/감사용 flow 로그 수집 후보 |
| R-013 | Calico 기본 네트워크 포트 | https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements | 2026-04-22 | latest docs | Calico VXLAN은 UDP 4789, Typha는 TCP 5473 사용 | NKS 기본 Calico 보안그룹 해석 기준 |
| R-014 | Cilium firewall rules | https://docs.cilium.io/en/stable/operations/system_requirements/ | 2026-04-22 | stable docs | Cilium VXLAN 기본 UDP 8472, health TCP 4240/ICMP, Hubble TCP 4244 필요 | NKS worker 보안그룹 추가/수정/삭제 판단 기준 |
| R-015 | chaining 관측 아키텍처 | https://docs.cilium.io/en/stable/installation/cni-chaining-generic-veth/ ; https://docs.cilium.io/en/stable/observability/metrics/ ; https://docs.cilium.io/en/stable/observability/hubble/configuration/export/ ; https://docs.tigera.io/calico/latest/networking/configuring/vxlan-ipip ; https://prometheus-operator.dev/docs/getting-started/introduction/ | 2026-05-06 | Cilium 1.19.x stable, Calico latest, Prometheus Operator docs | generic-veth chaining은 veth 기반 CNI 위에 Cilium을 붙이는 방식이며, Hubble metrics는 Prometheus/Grafana 시계열 관측, exporter는 flow 로그 출력에 사용됨. Prometheus Operator는 ServiceMonitor 기반 scrape 관리를 제공함. | `docs/01-observability-architecture-concepts.md` 이론 정리 근거 |
