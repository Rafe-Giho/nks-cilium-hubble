# 환경 인벤토리

- 목적: 현재 NKS 클러스터와 네트워크 관련 전제 조건을 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-05-06

## 클러스터 기본 정보

| 항목 | 값 |
| --- | --- |
| 프로젝트/테넌트 | `kubectl` 조회만으로 확인 불가 |
| 리전 | 판교. kubeconfig API endpoint 기준 `nks-kr1` |
| 클러스터명 | `ta-sgh-vxlan-sidecarless-cls` |
| kubeconfig 파일 | `/mnt/c/Users/user/Downloads/ta-sgh-vxlan-sidecarless-cls_kubeconfig.yaml` |
| kubeconfig context | `nks_ta-sgh-vxlan-sidecarless-cls_3103cceb-5512-4261-9d1e-21485d55eb1c` |
| Kubernetes 버전 | server `v1.33.4`, local kubectl client `v1.33.10` |
| 노드 OS/이미지 | `Ubuntu 22.04.5 LTS` |
| 노드 커널 | `5.15.0-164-generic` |
| 컨테이너 런타임 | `containerd://1.6.21` |
| 노드풀 구성 | `default-worker` node group, worker 2대 |
| 노드 | `ta-sgh-vxlan-sidecarless-cls-default-worker-node-0` / `node-1` |
| 노드 Internal IP | `192.168.200.71`, `192.168.200.105` |
| 퍼블릭/프라이빗 여부 | 노드 External-IP 없음. 실제 subnet/public-private 정책은 NKS 콘솔 확인 필요 |

## 접속 및 운영 정보

| 항목 | 값 |
| --- | --- |
| `kubectl` 접속 경로 | Ubuntu Linux 또는 WSL bash에서 kubeconfig 직접 지정 |
| API server | `https://3103cceb-nks-kr1.container.nhncloud.com:6443` |
| 접근 방식 | Ubuntu Linux 또는 WSL bash에서 `KUBECONFIG` 지정 후 접근 |
| 변경 가능 시간대 | PoC 기준 유연 |
| 운영 담당자 | 사용자 |
| 롤백 의사결정자 | 사용자 |

## 네트워크 및 애드온 현황

| 항목 | 값 |
| --- | --- |
| 현재 CNI 배포 형태 | NKS 기본 Calico VXLAN |
| Calico 버전 | `v3.30.2` |
| Calico backend | `vxlan` |
| Calico datastore | `etcdv3` |
| Calico eBPF | `FELIX_BPFENABLED=false` |
| Cilium 버전 | 미설치 |
| Hubble | 미설치 |
| NetworkPolicy 사용 여부 | 리소스 없음 |
| Ingress Controller | 없음 |
| 현재 Ingress 리소스 | 없음 |
| 현재 LoadBalancer Service | 없음 |
| LoadBalancer 사용 방식 | NKS 기본 LoadBalancer 사용 가능 여부는 샘플 Service로 재검증 필요 |
| 서비스 메시 사용 여부 | 미구축 |
| 관측 스택 | 기본 없음. Cilium chaining + Hubble, Pixie, Beyla 후보 검토 |
| 기본 Add-on | CoreDNS, metrics-server, OpenStack Cinder CSI |
| Pod CIDR | 노드 `.spec.podCIDR` 없음. 실제 Pod IP는 `10.100.x.x` Calico IPAM 할당 |
| Service CIDR | 관측된 Service IP 기준 `10.254.x.x` 대역 |
| metrics API | `v1beta1.metrics.k8s.io` Available True |
| Helm release | 없음 |
| CRD | 없음 |

## Calico CNI 기준

Calico DaemonSet 확인값:

```yaml
DATASTORE_TYPE: etcdv3
CALICO_NETWORKING_BACKEND: vxlan
FELIX_BPFENABLED: "false"
CNI_NET_DIR: /etc/cni/net.d/calico
CNI_CONF_NAME: 10-calico.conflist
```

실제 노드 CNI 파일:

```text
/etc/cni/net.d/calico/10-calico.conflist
/etc/cni/net.d/calico/calico-kubeconfig
/etc/cni/net.d/calico/calico-tls/
```

실제 CNI plugin 구성:

- `calico`
- `portmap`
- `bandwidth`

Cilium chaining 시 `cilium-cni`를 마지막 plugin으로 추가해야 합니다.

## 보안그룹 현황

사용자가 NKS 콘솔에서 확인한 기본 Calico worker 보안그룹 기준입니다.

| 항목 | 값 | 판단 |
| --- | --- | --- |
| worker self TCP 수신 | `TCP 1-65535` from worker security group | Hubble `4244`, Cilium health `4240` 포함 예상 |
| worker self UDP 수신 | `UDP 1-65535` from worker security group | Calico VXLAN `4789` 포함 |
| worker IPv4 송신 | 임의 포트 to `0.0.0.0/0` | worker 간/외부 송신 충족 예상 |
| Calico VXLAN | `UDP 4789` | 기존 NKS Calico용. 유지 |
| Calico Typha | `TCP 5473` | 기존 NKS Calico용. 유지 |
| NodePort | 수신 `TCP 30000-32767` from `0.0.0.0/0` | 샘플 Service/LoadBalancer 확인 전 유지 |

결론:

- 05번 실행 가이드에서는 Calico VXLAN을 유지하므로 Calico 관련 보안그룹 규칙 삭제 금지.
- Cilium chaining + Hubble은 worker self TCP 전체 허용 기준이면 기본 포트 충족 예상.
- 하드닝 전에는 `docs/04-nks-security-group.md` 기준으로 재검토 필요.

## 확인 필요 사항

- NKS 콘솔의 공식 Pod CIDR / Service CIDR 생성값
- NKS LoadBalancer 생성 가능 여부
- Hubble Relay/UI를 port-forward 외 방식으로 노출할지 여부
- 운영형 모니터링에서 Prometheus/Grafana/exporter까지 포함할지 여부
- Pixie/Beyla/Calico-eBPF 비교 검증 순서

## 설계 가정

- 실제 업무 워크로드 대신 샘플 HTTP 워크로드와 테스트 Pod로 가시성을 검증합니다.
- Cilium full replacement는 폐기하고, Calico 유지 기반 관측을 우선 검토합니다.
- 실패 시 일반 in-place 복구보다 PoC 클러스터 재생성을 기본 롤백 경로로 둡니다.

## 메모

- 현재 문서는 2026-05-06 `kubectl --kubeconfig` read-only 조회 결과를 기준으로 작성했습니다.
- Cilium/Hubble 설치 변경은 아직 수행하지 않았습니다.
