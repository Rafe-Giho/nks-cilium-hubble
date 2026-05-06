# NKS Security Group 사전점검

- 목적: NKS 기본 Calico 보안그룹을 Cilium full replacement PoC 당시 기준으로 평가하고 추가/수정/삭제 기준을 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-04-22

## 결론

사용자가 확인한 현재 NKS worker 보안그룹 기준으로는 Cilium PoC를 위해 즉시 추가해야 할 보안그룹 규칙은 없습니다.

판단 근거:

- 현재 worker 보안그룹에는 worker 보안그룹 self 원격으로 `TCP 1-65535` 수신 허용이 있습니다.
- 현재 worker 보안그룹에는 worker 보안그룹 self 원격으로 `UDP 1-65535` 수신 허용이 있습니다.
- 현재 worker 보안그룹에는 IPv4 전체 송신 허용이 있습니다.
- 따라서 Cilium VXLAN 기본 port `UDP 8472`, Hubble server `TCP 4244`, Cilium health `TCP 4240`은 worker node 간 통신 기준으로 이미 포함됩니다.

이 PoC에서 보안그룹 조치 결론:

| 구분 | 조치 | 이유 |
| --- | --- | --- |
| 추가 | 없음 | worker self `TCP/UDP 1-65535`가 Cilium 기본 포트를 포함 |
| 수정 | 없음 | NKS Calico VXLAN은 `4789`, Cilium VXLAN은 `8472`라 기본값끼리 충돌하지 않음 |
| 삭제 | 없음 | NKS 관리형 Calico add-on/부트스트랩 영향이 확인되지 않았으므로 PoC에서는 기존 Calico 관련 규칙을 삭제하지 않음 |

## 기본 포트 기준

| 구성 | 기본 포트 | 방향 | 필요 범위 |
| --- | --- | --- | --- |
| Calico VXLAN | `UDP 4789` | node 간 양방향 | 기존 NKS Calico용 |
| Calico Typha | `TCP 5473` | calico-node -> typha | 기존 NKS Calico용 |
| Cilium VXLAN | `UDP 8472` | node 간 양방향 | Cilium tunnel mode |
| Cilium Geneve | `UDP 6081` | node 간 양방향 | Geneve 사용 시만 |
| Cilium health | `TCP 4240`, ICMP | node 간 양방향 | 권장. 미허용 시 Cilium 자체 동작은 가능하지만 health 정보 제한 |
| Hubble server | `TCP 4244` | Hubble Relay -> Cilium agent | Hubble Relay 사용 시 필요 |
| Hubble Relay | `TCP 4245` | Hubble CLI/UI -> Relay | 기본은 ClusterIP/port-forward. 외부 노출 시 별도 보호 필요 |
| Cilium agent metrics | `TCP 9962` | Prometheus -> Cilium agent | metrics 사용 시 |
| Cilium operator metrics | `TCP 9963` | Prometheus -> operator | metrics 사용 시 |
| Cilium Envoy metrics | `TCP 9964` | Prometheus -> Envoy | Envoy metrics 사용 시 |
| Hubble metrics | `TCP 9965` | Prometheus -> Hubble metrics | Hubble metrics 사용 시 |
| Hubble Relay metrics | `TCP 9966` | Prometheus -> Hubble Relay | Relay metrics 사용 시 |

## 현재 NKS 기본 보안그룹 평가

사용자가 제공한 현재 NKS worker 보안그룹 중 Cilium 관점의 핵심 규칙은 아래입니다.

| 현재 규칙 | Cilium 영향 | 판단 |
| --- | --- | --- |
| 수신 `TCP 1-65535` from worker security group | `4240`, `4244`, `4245`, metrics TCP port 포함 | 충족 |
| 수신 `UDP 1-65535` from worker security group | `8472` VXLAN 포함 | 충족 |
| 송신 임의 IPv4 `0.0.0.0/0` | worker node에서 control plane, worker, 외부 endpoint로 송신 가능 | 충족 |
| 수신 `TCP 30000-32767` from `0.0.0.0/0` | NodePort/LB backend 경로 가능 | 유지 |
| 수신/송신 `UDP 4789` | 기존 Calico VXLAN | PoC 중 유지 |
| 수신/송신 `TCP 5473` | 기존 Calico Typha | PoC 중 유지 |
| NKS control plane 관련 `6443`, `10250`, `2379`, `5473` | NKS 관리형 제어면/노드 운영 | 유지 |

따라서 현재 상태에서는 Cilium 기본 VXLAN `UDP 8472`를 사용해도 별도 보안그룹 추가 없이 worker 간 overlay 통신이 가능해야 합니다.

## Archived full replacement 적용 기준

실패로 종결한 full replacement 실험에서는 Cilium VXLAN port를 기본값으로 뒀습니다.

```bash
export CILIUM_TUNNEL_PORT="8472"
```

현재 NKS Calico는 `UDP 4789`이고 Cilium VXLAN 기본은 `UDP 8472`라 기본값끼리 충돌하지 않습니다.

## 설치 전 확인

NKS 콘솔에서 아래 중 하나가 성립하는지 확인합니다.

권장 확인:

| 방향 | 프로토콜 | 포트 | 원격 |
| --- | --- | --- | --- |
| 수신 | TCP | `1-65535` | worker security group self |
| 수신 | UDP | `1-65535` | worker security group self |
| 송신 | 임의 | 전체 | IPv4 `0.0.0.0/0` 또는 worker/control-plane 대상 |

또는 하드닝된 최소 규칙:

| 방향 | 프로토콜 | 포트 | 원격 |
| --- | --- | --- | --- |
| 수신 | UDP | `8472` | worker security group self |
| 수신 | TCP | `4240` | worker security group self |
| 수신 | TCP | `4244` | worker security group self |
| 수신 | ICMP | echo request/reply | worker security group self |

metrics를 외부 Prometheus가 scrape한다면 아래를 Prometheus 위치에 맞게 제한 허용합니다.

| 방향 | 프로토콜 | 포트 | 원격 |
| --- | --- | --- | --- |
| 수신 | TCP | `9962-9966` | Prometheus node/security group/CIDR |

클러스터 내부 Prometheus만 사용할 경우에는 public CIDR에 metrics port를 열지 않습니다.

## 삭제하지 않는 규칙

PoC 중에는 아래 규칙을 삭제하지 않습니다.

- Calico VXLAN `UDP 4789`
- Calico Typha `TCP 5473`
- NKS control plane 관련 `6443`, `10250`, `2379`
- NodePort `30000-32767`

이유:

- NKS는 CNI 변경을 공식 지원하지 않습니다.
- NKS 관리형 Calico add-on 또는 노드 부트스트랩이 어떤 규칙과 리소스를 전제로 하는지 확정되지 않았습니다.
- PoC 실패 시 클러스터 재생성이 기본 롤백이므로, 보안그룹 삭제로 복구 변수를 늘리지 않습니다.

Calico 관련 규칙 삭제는 아래 조건이 모두 충족된 뒤 별도 변경으로만 검토합니다.

- 모든 신규 Pod가 Cilium CIDR을 사용
- 신규 scale-out node도 Cilium CNI를 사용
- NKS가 Calico add-on 또는 관련 보안그룹 규칙을 재생성하지 않는지 확인
- 롤백 전략이 클러스터 재생성 외에도 명확

## 참고 출처

- https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements
- https://docs.tigera.io/calico/latest/reference/felix/configuration
- https://docs.cilium.io/en/stable/operations/system_requirements/
- https://docs.cilium.io/en/stable/helm-reference/
