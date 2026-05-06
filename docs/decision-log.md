# 결정 로그

- 목적: 프로젝트 주요 판단과 근거를 추적합니다.
- 상태: draft
- 마지막 갱신: 2026-05-06

| ID | 날짜 | 주제 | 결정 | 근거 | 영향 | 상태 |
| --- | --- | --- | --- | --- | --- | --- |
| D-001 | 2026-04-16 | 작업 방식 | 실제 클러스터 변경 전에 근거 수집과 문서 정리를 먼저 수행 | 초기 하네스 합의 | 설계/검증 문서 우선 진행 | confirmed |
| D-002 | 2026-04-16 | 역할 분담 | 사용자가 클러스터 생성과 명령 실행을 수행하고, 저장소는 실행 방안/검증/문서화를 담당 | 사용자 요청 | 문서 중심 진행 | confirmed |
| D-003 | 2026-04-16 | PoC 범위 | 단일 NKS 클러스터에서 Cilium CNI, Cilium Ingress, Hubble Relay/UI/CLI까지 검증 | 사용자 범위 확정 | 실제 앱 연동은 제외 | confirmed |
| D-004 | 2026-04-16 | 검증 방식 | 실제 업무 앱 대신 샘플 워크로드 기반으로 가시성 검증 수행 | 현재 준비 가능한 자산 기준 | 샘플 manifest 필요 | confirmed |
| D-005 | 2026-04-22 | 비지원 PoC 수용 | NKS 공식 문서상 cluster CNI 변경 기능은 비지원 상태이므로, 이번 작업은 운영 적용이 아닌 기능 검증 PoC로 진행 | NHN Cloud NKS 릴리스 노트 | 실패 시 운영 롤백이 아닌 PoC 클러스터 재생성을 기본 복구 경로로 둠 | confirmed |
| D-006 | 2026-04-22 | 성공 판단 기준 | Hubble 가시성뿐 아니라 샘플 워크로드 기준 성능/운영 단순화 지표도 수집 | sidecar 오버헤드 대안 검토 목적 | 결과 검토 시 기능/성능/운영성 항목을 분리 평가 | confirmed |
| D-007 | 2026-04-22 | 클러스터 접근 원칙 | 클러스터에는 직접 변경을 하지 않고 read-only 정보 수집만 수행 | 사용자 요청 | 이후 조사 단계는 조회 중심으로 진행 | confirmed |
| D-008 | 2026-04-22 | 구현 우선순위 | 1순위 Cilium full replacement + Ingress, 2순위 Gateway API, 3순위 Calico chaining, 추가 후보 Pixie/Beyla | 사용자 요청 | Track A 실패로 후속 우선순위 재설계 필요 | superseded |
| D-009 | 2026-04-22 | 실행 가이드 구조 | 설치 문서를 번호형 실행 가이드로 재구성하고 Track A를 기본 실행 문서로 지정 | 사용자가 순서대로 따라 실행 가능한 가이드 요청 | Track A 문서는 아카이브로 전환, 후속 실행 가이드 재작성 필요 | superseded |
| D-010 | 2026-04-30 | NKS Cilium full replacement 운영화 후보 | Cilium PodCIDR은 기존 Calico 노드 블록을 재사용하고, control plane 반환 경로는 route keeper DaemonSet으로 보정 | `10.100.64.0/18` 하위 대역은 metrics APIService timeout, 기존 Calico 블록과 `vxlan.calico` 반환 route 복구 시 metrics APIService 성공 | scale-out/노드 재부팅/peer 변경 자동화가 남은 운영 리스크 | applied-in-poc |
| D-011 | 2026-04-30 | route keeper 방식의 프로젝트 적합성 | route keeper 방식은 Track A PoC 지속에는 유효하지만, 운영 단순화 목표에는 부적합에 가까운 보조 우회책으로 분류 | route keeper, Calico VXLAN 흔적, 하드코딩 peer가 추가 장애 지점과 운영 절차를 만듦 | 최종 후보 판단 시 Track C 또는 대안 sidecar-less 관측 도구와 비교 필요 | proposed |
| D-012 | 2026-04-30 | Track A full replacement 판정 | NKS에서 Cilium full replacement는 실패로 결론 내리고, 후속 설계는 Calico 유지/Calico-eBPF/sidecar-less 관측 도구 중심으로 재구성 | 새 Cilium CIDR과 NKS Pod CIDR 하위 대역 모두 metrics APIService 실패, 기존 Calico VXLAN 호환 경로와 route keeper가 있어야 정상화 | Track A는 보고서로 종결하고 후속 Track 재설계 필요 | concluded |
| D-013 | 2026-05-06 | 신규 클러스터 Track C 기준선 | `ta-sgh-vxlan-sidecarless-cls`에서는 NKS 기본 Calico VXLAN을 유지하고 Cilium은 generic-veth chaining + Hubble 용도로만 설치 | 새 클러스터가 Calico `v3.30.2`, `etcdv3`, `vxlan`, veth 기반 Pod 인터페이스, metrics API 정상 상태로 확인됨 | `docs/30-track-c-calico-chaining-hubble.md`와 `manifests/30-*`를 실행 기준으로 사용 | ready |
| D-014 | 2026-05-06 | Track C 기본 검증 결과 | Calico 유지 + Cilium generic-veth chaining + Hubble 기본 관측을 성공으로 판정 | 신규 `curl` Pod가 Calico IP를 받고 API/CoreDNS/metrics API 통신 성공, Hubble Relay healthcheck OK, connected nodes 2/2, `cilium-chain-check` DNS/CoreDNS metrics flow 확인 | 다음 단계는 Hubble CLI 버전 정합화, UI 접근 확인, 운영형 metrics/exporter 검토 | validated-basic |
| D-015 | 2026-05-06 | 운영형 Grafana 관측 방식 | Track C 운영형 대시보드는 `kube-prometheus-stack` + Cilium/Hubble `ServiceMonitor` + Grafana dashboard ConfigMap 방식으로 진행 | Hubble UI/CLI는 실시간 flow 확인에 적합하고, Grafana는 Prometheus 기반 시계열/알람/대시보드 운영에 적합함 | `docs/15-hubble-network-monitoring.md`, `manifests/15-*`를 Grafana 검증 기준으로 사용 | ready |
