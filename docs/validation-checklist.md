# 검증 체크리스트

- 목적: 변경 전후 확인 항목을 일관되게 관리합니다.
- 상태: draft
- 마지막 갱신: 2026-05-06

## 변경 전 확인

| 항목 | 기대 결과 | 상태 |
| --- | --- | --- |
| PoC 클러스터 재생성 기준 확인 | 실패 시 복구 경로 명확 | 확인: 신규 클러스터 `ta-sgh-vxlan-sidecarless-cls` |
| 클러스터가 생성 직후 기본 Calico 상태인지 확인 | 기준선 일치 | 통과 |
| worker node 수와 flavor 확인 | PoC 최소 조건 충족 | 2대 Ready, flavor는 콘솔 확인 필요 |
| Calico Pod 정상 | `calico-node` 2/2, typha/controller Ready | 통과 |
| metrics APIService 정상 | `v1beta1.metrics.k8s.io AVAILABLE=True` | 통과 |
| NetworkPolicy 없음 | 기존 정책 영향 없음 | 통과 |
| Helm release 없음 | 새 설치 기준선 | 통과 |
| CRD 없음 | Cilium 잔존 없음 | 통과 |
| 실제 Calico CNI 설정 확인 | `/etc/cni/net.d/calico/10-calico.conflist` 확인 | 통과 |
| generic-veth 가능 여부 확인 | 노드에 `cali...` veth 존재 | 통과 |
| NKS worker self TCP 전체 허용 확인 | `TCP 1-65535` from worker security group 유지 | 사용자 제공 기준 충족 |
| NKS worker self UDP 전체 허용 확인 | `UDP 1-65535` from worker security group 유지 | 사용자 제공 기준 충족 |
| Hubble Relay용 노드 간 4244/TCP 경로 검토 | Relay 통신 가능 | 사용자 제공 기준 충족 예상 |
| Cilium health용 4240/TCP와 ICMP 검토 | health 확인 가능 | TCP 충족 예상, ICMP 확인 필요 |

## Cilium Chaining 확인

| 항목 | 기대 결과 | 상태 |
| --- | --- | --- |
| `cni-configuration` ConfigMap 적용 | 실제 Calico etcd 기반 CNI + `cilium-cni` 포함 | 통과 |
| Cilium Helm release 설치 | `kube-system/cilium` deployed | 통과 |
| Cilium DaemonSet 정상 기동 | `ds/cilium` Ready 2/2 | 통과 |
| Cilium Operator 정상 | Deployment Ready | 통과: 2/2 |
| `cilium status --wait` 정상 | Cilium OK | 통과: 리소스 Ready, CLI 출력 별도 보존 필요 |
| Calico 기존 네트워킹 유지 | `calico-node` 2/2 유지 | 통과: 신규 Pod/Service 통신 정상 |
| kube-proxy replacement 비활성 | `kubeProxyReplacement=false` | 통과 |
| metrics APIService 유지 | Available True | 통과 |

## 신규 Pod 확인

| 항목 | 기대 결과 | 상태 |
| --- | --- | --- |
| `cilium-chain-check` namespace 생성 | namespace Ready | 통과 |
| 신규 `curl` Pod Ready | Calico IP 할당, Pod Ready | 통과: `10.100.188.10` |
| API Service DNS/통신 | `kubernetes.default.svc` 접근 가능 | 통과: `401 Unauthorized` 정상 응답 |
| kube-dns Service 통신 | `kube-dns.kube-system.svc.cluster.local:9153` 접근 가능 | 통과 |
| 기존 kube-system 영향 없음 | CoreDNS/metrics-server 정상 | 통과 |

## Hubble 확인

| 항목 | 기대 결과 | 상태 |
| --- | --- | --- |
| Hubble Relay 정상 기동 | Relay Ready | 통과: 1/1 |
| Hubble UI 정상 기동 | UI 접근 가능 | 부분 통과: Pod 2/2, 브라우저 접근 미확인 |
| Hubble CLI 접근 가능 | Relay 연동 가능 | 통과: `Healthcheck Ok`, `Connected Nodes 2/2` |
| 신규 namespace flow 조회 | `cilium-chain-check` flow 확인 | 통과 |
| DNS, Pod-to-Service 흐름 조회 가능 | 요구 가시성 확보 | 통과: `:53`, `:9153` flow 확인 |
| 네임스페이스 또는 라벨 기준 필터링 가능 | 운영 확인 가능 | 통과: namespace filter 확인 |
| Hubble metrics 활성화 여부 판단 | 운영형 모니터링 범위 결정 | 결정: Grafana용 Hubble metrics 우선, exporter는 후속 |
| Hubble exporter 사용 여부 판단 | 장기 flow 검색/감사 범위 결정 | `[입력 필요]` |

주의:

- `hubble-cli-version=1.18.6`, `hubble-relay-version=1.19.3` 경고 확인. 장기 검증 전 CLI 업그레이드 권장.
- port-forward 세션이 `connection reset by peer`, `EOF`, `connection refused`로 종료될 수 있음. Hubble 실패가 아니라 port-forward 세션 종료로 분류.
- `Unsupported L3 protocol DROPPED (ICMPv6 RouterSolicitation)`은 IPv6 비활성 환경의 비차단 이벤트로 분류.

## Grafana 운영형 확인

| 항목 | 기대 결과 | 상태 |
| --- | --- | --- |
| kube-prometheus-stack values 검증 | Helm template 성공 | 통과: 로컬 template 검증 |
| Cilium/Hubble metrics values 검증 | 05번 chaining 설정 유지 + ServiceMonitor/dashboard 렌더링 | 통과: 로컬 template 검증 |
| kube-prometheus-stack 설치 | `monitoring` namespace 주요 Pod Ready | `[입력 필요]` |
| Cilium/Hubble metrics 적용 | `hubble-metrics` Service와 `ServiceMonitor` 생성 | `[입력 필요]` |
| Prometheus target 확인 | Cilium/Hubble target `UP` | `[입력 필요]` |
| Grafana dashboard 확인 | Cilium/Hubble dashboard 자동 로드 | `[입력 필요]` |
| PromQL 확인 | `{__name__=~"hubble_.*"}` 결과 존재 | `[입력 필요]` |

## 대안 관측 도구 확인

| 항목 | 기대 결과 | 상태 |
| --- | --- | --- |
| Pixie 검토 | CNI 변경 없는 eBPF 관측 가능성 판단 | `[입력 필요]` |
| Grafana Beyla 검토 | 앱/서비스 관측 가능성 판단 | `[입력 필요]` |
| Calico-eBPF 검토 | NKS 적용 가능성 별도 판단 | `[입력 필요]` |

## 성능/운영성 확인

| 항목 | 기대 결과 | 상태 |
| --- | --- | --- |
| 샘플 HTTP 요청 p50/p95 지연 수집 | 후속 비교 기준 확보 | `[입력 필요]` |
| 샘플 backend Pod CPU/메모리 수집 | 워크로드 기준선 확보 | `[입력 필요]` |
| Cilium/Hubble Pod CPU/메모리 수집 | 관측 구성 오버헤드 확인 | `[입력 필요]` |
| 설치/점검 단계 수 기록 | 운영 단순화 판단 근거 확보 | `[입력 필요]` |
| Hubble metrics 또는 exporter 선택 결과 기록 | PoC/운영 모니터링 경계 명확화 | `[입력 필요]` |

## 롤백 조건

- 신규 Pod 생성 실패
- kube-system 주요 Pod 비정상
- metrics APIService 비정상화
- Calico CNI 기본 네트워킹 장애
- Hubble 요구사항 미충족과 동시에 복구 불가

## 롤백 방식

- 1차: `helm -n kube-system uninstall cilium`
- 2차: `cni-configuration` ConfigMap 삭제
- 3차: 노드의 `/etc/cni/net.d/calico/05-cilium.conflist` 잔존 시 제거
- 최종: PoC 클러스터 삭제 후 재생성
