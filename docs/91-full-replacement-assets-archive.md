# Cilium Full Replacement 자산 보존

- 목적: Cilium full replacement 실험 중 작성·적용한 values, manifest, script, 주요 파일 경로를 보존합니다.
- 상태: archived
- 마지막 갱신: 2026-04-30

## 보존 원칙

이 문서는 실험 자산의 내용을 요약 보존합니다. 원본 manifest, script, artifact 로그는 후속 재설계에 사용하지 않으므로 삭제했고, 필요한 핵심 내용만 이 문서에 남깁니다.

## Manifest/values 목록

아래 파일은 삭제된 원본 파일명입니다. 핵심 내용은 이 문서에 보존합니다.

| 파일 | 목적 | 상태 |
| --- | --- | --- |
| `manifests/10-cilium-values-initial.yaml` | `10.120.0.0/16` 신규 Cilium CIDR 실험 | 실패 기록 |
| `manifests/10-cilium-values-initial.generated.yaml` | Cilium CLI dry-run 생성값 | 실패 기록 |
| `manifests/12-cilium-values-nks-pod-cidr.yaml` | `10.100.64.0/18` NKS Pod CIDR 하위 대역 실험 | 실패 기록 |
| `manifests/12-cilium-values-nks-pod-cidr.generated.yaml` | Cilium CLI dry-run 생성값 | 실패 기록 |
| `manifests/13-cilium-values-known-calico-blocks.yaml` | 기존 Calico 블록 재사용 실험 | 우회 성공 |
| `manifests/13-cilium-values-known-calico-blocks.generated.yaml` | Cilium CLI dry-run 생성값 | 우회 성공 |
| `manifests/14-nks-cilium-control-plane-route-keeper.yaml` | control plane 반환 route keeper DaemonSet | 우회 성공 |

## Script 목록

아래 파일은 삭제된 원본 파일명입니다. 핵심 명령은 이 문서에 보존합니다.

| 파일 | 목적 | 상태 |
| --- | --- | --- |
| `scripts/inplace-cilium-cidr-switch.sh` | Cilium PodCIDR을 `10.100.64.0/18`로 in-place 변경 | 실패 기록 |
| `scripts/inplace-cilium-known-calico-blocks.sh` | 기존 Calico 블록을 Cilium cluster-pool로 재사용 | 우회 성공 |
| `scripts/fix-control-plane-return-routes.sh` | control plane 반환 route 수동 복구 | 우회 성공 |
| `scripts/packet-check-metrics.sh` | metrics-server 통신 tcpdump 확인 | 원인 분석 |
| `scripts/final-check-cilium-known-blocks.sh` | 기존 Calico 블록 구성 최종 점검 | 검증 |
| `scripts/verify-route-keeper.sh` | route keeper와 metrics API 검증 | 검증 |
| `scripts/reboot-node-verify-route-keeper.sh` | 노드 재부팅 후 복구 검증 | 검증 |

## 핵심 values

### `10-cilium-values-initial.yaml`

```yaml
operator:
  unmanagedPodWatcher:
    restart: false
cluster:
  name: nks-cilium-hubble
agentNotReadyTaintKey: node.cilium.io/agent-not-ready
tolerations:
  - operator: Exists
routingMode: tunnel
tunnelProtocol: vxlan
tunnelPort: 8472
cni:
  customConf: true
  uninstall: false
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
      - "10.120.0.0/16"
policyEnforcementMode: never
bpf:
  hostLegacyRouting: true
hubble:
  relay:
    enabled: true
  ui:
    enabled: true
```

### `12-cilium-values-nks-pod-cidr.yaml`

```yaml
operator:
  unmanagedPodWatcher:
    restart: false
cluster:
  name: nks-cilium-hubble
agentNotReadyTaintKey: node.cilium.io/agent-not-ready
tolerations:
  - operator: Exists
routingMode: tunnel
tunnelProtocol: vxlan
tunnelPort: 8472
cni:
  customConf: true
  uninstall: false
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
      - "10.100.64.0/18"
    clusterPoolIPv4MaskSize: 24
policyEnforcementMode: never
bpf:
  hostLegacyRouting: true
hubble:
  relay:
    enabled: true
  ui:
    enabled: true
```

### `13-cilium-values-known-calico-blocks.yaml`

```yaml
operator:
  unmanagedPodWatcher:
    restart: false
cluster:
  name: nks-cilium-hubble
agentNotReadyTaintKey: node.cilium.io/agent-not-ready
tolerations:
  - operator: Exists
routingMode: tunnel
tunnelProtocol: vxlan
tunnelPort: 8472
cni:
  customConf: true
  uninstall: false
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
      - "10.100.188.0/24"
      - "10.100.145.0/24"
    clusterPoolIPv4MaskSize: 24
policyEnforcementMode: never
bpf:
  hostLegacyRouting: true
hubble:
  relay:
    enabled: true
  ui:
    enabled: true
```

## Route keeper 핵심 manifest

적용 파일:

```text
manifests/14-nks-cilium-control-plane-route-keeper.yaml
```

원본 manifest는 삭제했습니다. 아래 설정은 삭제 전 적용했던 핵심값입니다.

핵심 환경변수:

```yaml
- name: UNDERLAY_DEV
  value: eth0
- name: INTERVAL_SECONDS
  value: "10"
- name: LOCAL_VXLAN_MAC_MAP
  value: "ta-sgh-cilium-hubble-cls-default-worker-node-0=66:40:d6:3d:2f:56,ta-sgh-cilium-hubble-cls-default-worker-node-3=66:25:d5:b0:bd:0b"
- name: CONTROL_PLANE_ROUTES
  value: "10.100.39.0/24=10.100.39.0=66:82:0c:c7:49:56=192.168.200.55,10.100.155.0/24=10.100.155.0=66:a0:7e:c8:f6:82=192.168.200.43,10.100.160.0/24=10.100.160.0=66:5a:d8:ad:2a:40=192.168.200.100"
```

route keeper 스크립트의 핵심 동작:

```bash
bridge fdb replace "${mac}" dev "${VXLAN_DEV}" dst "${peer_ip}" self permanent
ip neigh replace "${gateway}" lladdr "${mac}" dev "${VXLAN_DEV}" nud permanent
ip route replace "${gateway}" dev "${VXLAN_DEV}" scope link
ip route replace "${cidr}" via "${gateway}" dev "${VXLAN_DEV}" onlink
```

interface 재생성 핵심:

```bash
ip link add "${VXLAN_DEV}" type vxlan id "${VXLAN_ID}" local "${NODE_IP}" dev "${UNDERLAY_DEV}" dstport "${VXLAN_PORT}" nolearning
ip link set dev "${VXLAN_DEV}" address "${local_mac}"
ip link set dev "${VXLAN_DEV}" mtu "${VXLAN_MTU}" up
```

## Script 핵심 명령 보존

### `inplace-cilium-cidr-switch.sh`

목적:

```text
Cilium PodCIDR을 10.100.64.0/18로 바꿔 NKS 원래 Pod CIDR 하위 대역 재사용 가능성을 검증.
```

핵심 명령:

```bash
cilium install \
  --version "${CILIUM_VERSION}" \
  --values "${CILIUM_VALUES}" \
  --dry-run-helm-values \
  > "${CILIUM_GENERATED_VALUES}"

helm upgrade --install cilium oci://quay.io/cilium/charts/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --values "${CILIUM_GENERATED_VALUES}"

kubectl delete ciliumendpoints.cilium.io --all -A --ignore-not-found
kubectl delete ciliumnodes.cilium.io --all --ignore-not-found
kubectl -n kube-system rollout restart ds/cilium
```

결과:

```text
Cilium 정상, metrics.k8s.io 실패.
```

### `inplace-cilium-known-calico-blocks.sh`

목적:

```text
기존 Calico 블록 10.100.188.0/24, 10.100.145.0/24를 Cilium cluster-pool로 재사용.
```

핵심 명령:

```bash
kubectl -n kube-system patch ds calico-node --type merge -p \
  '{"spec":{"template":{"spec":{"nodeSelector":{"io.cilium/disable-calico-node":"true"}}}}}'

kubectl -n kube-system delete pod -l k8s-app=calico-node --ignore-not-found

helm upgrade --install cilium oci://quay.io/cilium/charts/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --values "${CILIUM_GENERATED_VALUES}"

kubectl delete ciliumendpoints.cilium.io --all -A --ignore-not-found
kubectl delete ciliumnodes.cilium.io --all --ignore-not-found
kubectl -n kube-system rollout restart ds/cilium
```

결과:

```text
Cilium 정상, metrics.k8s.io는 route 복구 전까지 실패.
```

### `fix-control-plane-return-routes.sh`

목적:

```text
control plane 출발 대역 반환 route를 vxlan.calico로 수동 복구.
```

핵심 명령:

```bash
for b in 10.100.39.0 10.100.155.0 10.100.160.0; do
  ip route replace "$b" dev vxlan.calico scope link
  ip route replace "$b/24" via "$b" dev vxlan.calico onlink
done
```

결과:

```text
metrics.k8s.io 정상화.
```

### `packet-check-metrics.sh`

목적:

```text
metrics-server APIService timeout 구간의 실제 패킷 경로 확인.
```

핵심 명령:

```bash
kubectl -n kube-system run sniff-node0 --image=ghcr.io/nicolaka/netshoot:v0.13 --overrides='...' -- sleep 3600
kubectl -n kube-system run sniff-node3 --image=ghcr.io/nicolaka/netshoot:v0.13 --overrides='...' -- sleep 3600
kubectl -n kube-system exec sniff-node3 -- tcpdump -ni any host "${METRICS_IP}" and port 443
kubectl get --raw /apis/metrics.k8s.io/v1beta1
```

관측:

```text
vxlan.calico In/Out 경로가 있어야 metrics-server 응답이 돌아감.
```

### `final-check-cilium-known-blocks.sh`

목적:

```text
기존 Calico 블록 재사용 구성의 최종 상태 수집.
```

핵심 명령:

```bash
cilium status
kubectl get pod -A -o wide
kubectl get node -L io.cilium.migration/cilium-default -o wide
kubectl get ciliumnodes -o wide
kubectl get ciliumendpoints -A -o wide
kubectl get apiservice v1beta1.metrics.k8s.io -o wide
kubectl get --raw /apis/metrics.k8s.io/v1beta1
kubectl top node
```

### `verify-route-keeper.sh`

목적:

```text
route keeper DaemonSet 적용 후 route/FDB/neighbor와 metrics API 상태 확인.
```

핵심 명령:

```bash
kubectl -n kube-system get ds nks-cilium-control-plane-route-keeper -o wide
kubectl -n kube-system get pod -l app=nks-cilium-control-plane-route-keeper -o wide
kubectl -n kube-system logs -l app=nks-cilium-control-plane-route-keeper --tail=120

ip -d link show vxlan.calico
bridge fdb show dev vxlan.calico
ip neigh show dev vxlan.calico
ip -4 route | grep -E "10\.100\.(39|155|160)\.0"

kubectl get apiservice v1beta1.metrics.k8s.io -o wide
kubectl get --raw /apis/metrics.k8s.io/v1beta1
kubectl top node
```

self-heal 테스트:

```bash
ip route del 10.100.39.0/24
ip route del 10.100.39.0
sleep 15
ip route show 10.100.39.0
ip route show 10.100.39.0/24
```

### `reboot-node-verify-route-keeper.sh`

목적:

```text
노드 재부팅 후 Cilium, route keeper, metrics API가 복구되는지 확인.
```

핵심 명령:

```bash
kubectl debug "node/${NODE}" \
  --profile=sysadmin \
  --image=busybox:1.36 \
  -- chroot /host /sbin/reboot

kubectl wait --for=condition=Ready "node/${NODE}" --timeout=15m
kubectl -n kube-system rollout status ds/nks-cilium-control-plane-route-keeper --timeout=5m
kubectl -n kube-system rollout status ds/cilium --timeout=10m
cilium status --wait
SELF_HEAL_TEST=false bash scripts/verify-route-keeper.sh
```

## 삭제한 구문서와 대체 관계

아래 문서들은 이 보존 문서와 `docs/90-full-replacement-execution-record.md`로 대체합니다.

| 삭제 문서 | 대체 위치 |
| --- | --- |
| `docs/10-track-a-full-replacement-ingress.md` | `docs/90-full-replacement-execution-record.md` |
| `docs/11-track-a-copy-paste-runbook.md` | `docs/90-full-replacement-execution-record.md` |
| `docs/12-track-a-fresh-nks-pod-cidr-runbook.md` | `docs/90-full-replacement-execution-record.md` |
| `docs/13-track-a-known-calico-blocks-route-keeper.md` | `docs/90-full-replacement-execution-record.md`, `docs/91-full-replacement-assets-archive.md` |
| `docs/14-track-a-full-replacement-failure-report.md` | `docs/90-full-replacement-execution-record.md` |
| `docs/20-track-b-full-replacement-gateway-api.md` | 삭제. full replacement 전제 자체가 실패 |

## 주의

이 자산들은 후속 재설계에서 그대로 사용하지 않습니다.

필요한 경우에만 실패 재현, 원인 분석, 보고 근거 확인 용도로 참조합니다.
