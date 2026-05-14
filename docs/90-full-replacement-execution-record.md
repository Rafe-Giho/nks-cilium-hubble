# Cilium Full Replacement 실행 기록

- 목적: NKS에서 Calico를 Cilium으로 완전 대체하려고 수행한 명령, 구성 변경, 검증 결과를 보존합니다.
- 상태: archived
- 마지막 갱신: 2026-04-30

## 최종 판정

NKS에서 Cilium full replacement는 실패로 결론 내렸습니다.

성공한 최종 상태는 Cilium 단독 구성이 아니라 아래 우회 구성입니다.

```text
Cilium 주 CNI 전환 + Calico VXLAN 호환 경로 유지 + route keeper 보정
```

완전 대체 실패 근거:

```text
- 새 Cilium CIDR 10.120.0.0/16: metrics.k8s.io FailedDiscoveryCheck
- NKS 기존 Pod CIDR 하위 대역 10.100.64.0/18: metrics.k8s.io FailedDiscoveryCheck
- 기존 Calico 블록 10.100.188.0/24, 10.100.145.0/24: route keeper 추가 후에만 정상
- Calico VXLAN 흔적, control plane peer, vxlan.calico route가 필요
- Calico 완전 삭제 불가
```

## 현재 live 구성

2026-04-30 최종 확인 기준입니다.

```text
Cilium: OK
Cilium Envoy: OK
Hubble Relay: OK
Cluster Pods: 6/6 managed by Cilium
metrics.k8s.io: AVAILABLE=True
route keeper: 2/2 Ready
calico-node: Desired 0
calico-typha/calico-kube-controllers: Running
```

현재 Cilium Helm values:

```yaml
agentNotReadyTaintKey: node.cilium.io/agent-not-ready
bpf:
  hostLegacyRouting: true
cluster:
  name: nks-cilium-hubble
cni:
  customConf: true
  uninstall: false
hubble:
  relay:
    enabled: true
  ui:
    enabled: true
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4MaskSize: 24
    clusterPoolIPv4PodCIDRList:
    - 10.100.188.0/24
    - 10.100.145.0/24
k8sServiceHost: 7e449296-nks-kr1.container.nhncloud.com
k8sServicePort: 6443
kubeProxyReplacement: true
operator:
  replicas: 1
  unmanagedPodWatcher:
    restart: false
policyEnforcementMode: never
routingMode: tunnel
tolerations:
- operator: Exists
tunnelPort: 8472
tunnelProtocol: vxlan
```

현재 CiliumNode:

```text
ta-sgh-cilium-hubble-cls-default-worker-node-0   10.100.188.10   192.168.200.57
ta-sgh-cilium-hubble-cls-default-worker-node-3   10.100.145.67   192.168.200.53
```

현재 control plane 반환 경로:

| CIDR | gateway | peer MAC | peer underlay IP |
| --- | --- | --- | --- |
| `10.100.39.0/24` | `10.100.39.0` | `66:82:0c:c7:49:56` | `192.168.200.55` |
| `10.100.155.0/24` | `10.100.155.0` | `66:a0:7e:c8:f6:82` | `192.168.200.43` |
| `10.100.160.0/24` | `10.100.160.0` | `66:5a:d8:ad:2a:40` | `192.168.200.100` |

현재 route keeper 핵심 설정:

```text
image: ghcr.io/nicolaka/netshoot:v0.13
hostNetwork: true
privileged: true
capabilities: NET_ADMIN
interval: 10s
LOCAL_VXLAN_MAC_MAP:
  node-0=66:40:d6:3d:2f:56
  node-3=66:25:d5:b0:bd:0b
CONTROL_PLANE_ROUTES:
  10.100.39.0/24=10.100.39.0=66:82:0c:c7:49:56=192.168.200.55
  10.100.155.0/24=10.100.155.0=66:a0:7e:c8:f6:82=192.168.200.43
  10.100.160.0/24=10.100.160.0=66:5a:d8:ad:2a:40=192.168.200.100
```

## 실행 타임라인

### 1. Preflight 스냅샷

```bash
mkdir -p artifacts/preflight

kubectl config current-context | tee artifacts/preflight/context.txt
kubectl version -o yaml | tee artifacts/preflight/kubectl-version.yaml
kubectl get no -o wide | tee artifacts/preflight/nodes.txt
kubectl get pod -A -o wide | tee artifacts/preflight/pods.txt
kubectl get svc -A | tee artifacts/preflight/services.txt
kubectl get ingress -A | tee artifacts/preflight/ingress.txt
kubectl get networkpolicy -A | tee artifacts/preflight/networkpolicy.txt
kubectl -n kube-system get cm calico-config -o yaml | tee artifacts/preflight/calico-config.yaml
kubectl get no -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}' | tee artifacts/preflight/node-taints.txt
```

### 2. 최초 Cilium values 생성

초기 목표 CIDR은 기존 Calico와 겹치지 않는 `10.120.0.0/16`이었습니다.

```bash
export CILIUM_VERSION="1.19.3"
export CILIUM_CLUSTER_NAME="nks-cilium-hubble"
export CILIUM_POD_CIDR="10.120.0.0/16"
export CILIUM_TUNNEL_PORT="8472"
export CILIUM_AGENT_NOT_READY_TAINT_KEY="node.cilium.io/agent-not-ready"

cilium install \
  --version "${CILIUM_VERSION}" \
  --values manifests/10-cilium-values-initial.yaml \
  --dry-run-helm-values \
  > manifests/10-cilium-values-initial.generated.yaml
```

설치:

```bash
helm upgrade --install cilium oci://quay.io/cilium/charts/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --values manifests/10-cilium-values-initial.generated.yaml

cilium status --wait
kubectl -n kube-system get pod -l k8s-app=cilium -o wide
kubectl -n kube-system get deploy,ds,svc | grep -E 'cilium|hubble'
```

### 3. CiliumNodeConfig와 CNI 경로 보정

NKS kubelet은 CNI 설정을 `/etc/cni/net.d/calico` 경로에서 사용했습니다. 그래서 Cilium CNI 파일도 해당 경로 아래에 작성해야 했습니다.

```bash
cat <<'EOF' | kubectl apply --server-side -f -
apiVersion: cilium.io/v2
kind: CiliumNodeConfig
metadata:
  namespace: kube-system
  name: cilium-default
spec:
  nodeSelector:
    matchLabels:
      io.cilium.migration/cilium-default: "true"
  defaults:
    write-cni-conf-when-ready: /host/etc/cni/net.d/calico/05-cilium.conflist
    custom-cni-conf: "false"
    cni-chaining-mode: "none"
    cni-exclusive: "true"
EOF
```

잘못된 경로로 생성된 경우 아래로 보정했습니다.

```bash
kubectl -n kube-system patch ciliumnodeconfig cilium-default --type merge -p \
'{"spec":{"defaults":{"write-cni-conf-when-ready":"/host/etc/cni/net.d/calico/05-cilium.conflist"}}}'
```

노드별 CNI 파일 확인:

```bash
export CILIUM_POD="$(kubectl -n kube-system get pod \
  --field-selector spec.nodeName="${NODE}" \
  -l k8s-app=cilium \
  -o jsonpath='{.items[0].metadata.name}')"

kubectl -n kube-system exec "${CILIUM_POD}" -c cilium-agent -- \
  ls -la /host/etc/cni/net.d/calico

kubectl -n kube-system exec "${CILIUM_POD}" -c cilium-agent -- \
  cat /host/etc/cni/net.d/calico/05-cilium.conflist
```

### 4. 노드별 migration

사용한 기본 절차:

```bash
kubectl cordon "${NODE}"
kubectl drain "${NODE}" --ignore-daemonsets --delete-emptydir-data
kubectl label node "${NODE}" --overwrite io.cilium.migration/cilium-default=true

kubectl -n kube-system delete pod \
  --field-selector spec.nodeName="${NODE}" \
  -l k8s-app=cilium

kubectl -n kube-system rollout status ds/cilium --timeout=10m
```

재부팅 후 검증:

```bash
kubectl wait --for=condition=Ready node/"${NODE}" --timeout=10m
cilium status --wait

kubectl -n kube-system run verify-cilium-node \
  --restart=Never \
  --overrides="{\"spec\":{\"nodeName\":\"${NODE}\",\"tolerations\":[{\"operator\":\"Exists\"}]}}" \
  --image=busybox:1.36 \
  -- sleep 3600

kubectl -n kube-system wait --for=condition=Ready pod/verify-cilium-node --timeout=2m
kubectl -n kube-system get pod verify-cilium-node -o wide
kubectl -n kube-system delete pod verify-cilium-node
kubectl uncordon "${NODE}"
```

결과:

```text
신규 Pod가 Cilium CIDR을 받는 것까지는 성공.
그러나 metrics.k8s.io APIService는 실패.
```

### 5. CoreDNS, metrics-server, 샘플 Pod 검증

```bash
kubectl -n kube-system rollout restart deploy/coredns
kubectl -n kube-system rollout status deploy/coredns --timeout=3m
kubectl -n kube-system get pod -l k8s-app=kube-dns -o wide
cilium status

kubectl create namespace cilium-check --dry-run=client -o yaml | kubectl apply -f -
kubectl -n cilium-check run ip-check \
  --image=busybox:1.36 \
  --restart=Never \
  -- sleep 3600
kubectl -n cilium-check wait --for=condition=Ready pod/ip-check --timeout=3m
kubectl -n cilium-check get pod ip-check -o wide

kubectl run dns-check --rm -it --restart=Never \
  --image=busybox:1.36 \
  -- nslookup kubernetes.default.svc.cluster.local
```

`cilium-check` namespace 삭제 중 `metrics.k8s.io` discovery 실패 때문에 Terminating이 지연됐습니다.

확인 명령:

```bash
kubectl get ns cilium-check -o yaml
kubectl get apiservice v1beta1.metrics.k8s.io -o wide
kubectl describe apiservice v1beta1.metrics.k8s.io
kubectl get --raw /apis/metrics.k8s.io/v1beta1
```

### 6. metrics-server 진단

```bash
kubectl get apiservice v1beta1.metrics.k8s.io -o wide
kubectl get pod -n kube-system -l k8s-app=metrics-server -o wide
kubectl logs -n kube-system -l k8s-app=metrics-server --tail=100
kubectl describe apiservice v1beta1.metrics.k8s.io
kubectl -n kube-system get svc,endpoints,endpointslice -l k8s-app=metrics-server -o wide
kubectl -n kube-system get deploy metrics-server -o yaml | grep -E 'hostNetwork|secure-port|kubelet|--'
kubectl get --raw /apis/metrics.k8s.io/v1beta1
```

노드 kubelet 10250 접근 확인:

```bash
kubectl -n kube-system run test-node10250 \
  --restart=Never \
  --rm -it \
  --image=ghcr.io/nicolaka/netshoot:v0.13 \
  -- /bin/bash -c 'nc -vz 192.168.200.57 10250; nc -vz 192.168.200.53 10250'
```

판정:

```text
metrics-server 자체나 kubelet 접근 문제가 아니라 kube-apiserver -> metrics-server Pod IP 경로 문제.
```

### 7. `10.100.64.0/18` 재시도

기존 NKS Pod CIDR 안의 하위 대역이면 control plane 라우팅이 될지 확인했습니다.

실행 스크립트:

```bash
bash scripts/inplace-cilium-cidr-switch.sh
```

주요 처리:

```text
- Cilium Helm values를 10.100.64.0/18로 변경
- CiliumNode/CiliumEndpoint 재생성
- kube-system workload 재시작
- metrics APIService 재확인
```

결과:

```text
v1beta1.metrics.k8s.io   kube-system/metrics-server   False (FailedDiscoveryCheck)
```

artifact:

```text
artifacts/inplace-cidr-switch-20260430-131624
```

판정:

```text
NKS control plane은 10.100.0.0/16 집계 CIDR만으로 Pod 대역 전체를 라우팅하는 구조가 아님.
```

### 8. 기존 Calico 블록 재사용

기존 Calico가 실제 사용하던 블록을 Cilium cluster-pool로 재사용했습니다.

실행 스크립트:

```bash
bash scripts/inplace-cilium-known-calico-blocks.sh
```

주요 처리:

```text
- calico-node DaemonSet nodeSelector를 io.cilium/disable-calico-node=true로 변경
- calico-node Pod 제거
- stale Calico route 제거
- Cilium PodCIDR을 10.100.188.0/24, 10.100.145.0/24로 재설정
- CiliumNode/CiliumEndpoint 재생성
- kube-system workload 재시작
```

초기 결과:

```text
Cilium 자체는 OK.
metrics.k8s.io는 여전히 FailedDiscoveryCheck.
```

artifact:

```text
artifacts/inplace-known-calico-blocks-20260430-133656
```

### 9. 패킷 확인

```bash
bash scripts/packet-check-metrics.sh
```

관측 결과:

```text
vxlan.calico In  10.100.160.0 -> 10.100.145.2:443
vxlan.calico Out 10.100.145.2:443 -> 10.100.160.0
```

artifact:

```text
artifacts/packet-check-20260430-134222
```

판정:

```text
kube-apiserver/control plane 요청은 worker node까지 도달함.
실패 원인은 metrics-server Pod 응답을 control plane 출발 대역으로 돌려보내는 반환 route/FDB/neighbor 상태.
```

### 10. control plane 반환 route 수동 복구

```bash
bash scripts/fix-control-plane-return-routes.sh
```

적용 route:

```bash
ip route replace 10.100.39.0 dev vxlan.calico scope link
ip route replace 10.100.39.0/24 via 10.100.39.0 dev vxlan.calico onlink
ip route replace 10.100.155.0 dev vxlan.calico scope link
ip route replace 10.100.155.0/24 via 10.100.155.0 dev vxlan.calico onlink
ip route replace 10.100.160.0 dev vxlan.calico scope link
ip route replace 10.100.160.0/24 via 10.100.160.0 dev vxlan.calico onlink
```

결과:

```bash
kubectl get --raw /apis/metrics.k8s.io/v1beta1
kubectl get apiservice v1beta1.metrics.k8s.io -o wide
kubectl top node
```

정상화:

```text
v1beta1.metrics.k8s.io   kube-system/metrics-server   True
```

### 11. route keeper 적용

수동 route 복구를 DaemonSet으로 전환했습니다.

```bash
kubectl apply -f manifests/14-nks-cilium-control-plane-route-keeper.yaml
kubectl -n kube-system rollout status ds/nks-cilium-control-plane-route-keeper --timeout=3m
kubectl -n kube-system get pod -l app=nks-cilium-control-plane-route-keeper -o wide
```

self-heal 검증:

```bash
SELF_HEAL_TEST=true bash scripts/verify-route-keeper.sh
```

결과:

```text
10.100.39.0 dev vxlan.calico scope link
10.100.39.0/24 via 10.100.39.0 dev vxlan.calico onlink
```

artifact:

```text
artifacts/route-keeper-verify-20260430-135454
```

### 12. 노드 재부팅 검증

node-0:

```bash
bash scripts/reboot-node-verify-route-keeper.sh ta-sgh-cilium-hubble-cls-default-worker-node-0
SELF_HEAL_TEST=false bash scripts/verify-route-keeper.sh
```

node-3:

```bash
kubectl wait --for=condition=Ready node/ta-sgh-cilium-hubble-cls-default-worker-node-3 --timeout=15m
SELF_HEAL_TEST=false bash scripts/verify-route-keeper.sh
```

결과:

```text
node-0 재부팅 후 Cilium/route keeper/metrics 복구 성공
node-3 재부팅 후 Cilium/route keeper/metrics 복구 성공
```

artifact:

```text
artifacts/reboot-route-keeper-20260430-135813
artifacts/route-keeper-verify-20260430-140139
artifacts/route-keeper-verify-20260430-142823
```

## 결과 매트릭스

| 시도 | Cilium 상태 | metrics.k8s.io | Calico 의존 | 판정 |
| --- | --- | --- | --- | --- |
| `10.120.0.0/16` 신규 CIDR | OK | 실패 | 없음 | 실패 |
| `10.100.64.0/18` NKS Pod CIDR 하위 대역 | OK | 실패 | 없음 | 실패 |
| 기존 Calico 블록 재사용 | OK | 초기 실패 | 있음 | 부분 성공 |
| 기존 Calico 블록 + route keeper | OK | 성공 | 있음 | 우회 성공, 완전 대체 아님 |

## 삭제 전 artifact 목록

```text
artifacts/preflight/
artifacts/inplace-cidr-switch-20260430-131624/
artifacts/inplace-known-calico-blocks-20260430-133656/
artifacts/packet-check-20260430-134222/
artifacts/final-check-20260430-134543/
artifacts/route-keeper-verify-20260430-135454/
artifacts/reboot-route-keeper-20260430-135813/
artifacts/route-keeper-verify-20260430-140139/
artifacts/route-keeper-verify-20260430-142823/
```

원본 artifact 로그는 full replacement 실패 보고 내용과 핵심 명령을 이 문서와 `docs/91-full-replacement-assets-archive.md`에 옮긴 뒤 삭제했습니다. 위 목록은 실행 증적의 디렉터리명 기록입니다.

## 최종 결론

이 기록은 재현을 위한 운영 절차가 아니라 실패한 full replacement 실험의 보존 기록입니다.

후속 설계에서는 이 방식을 운영 후보로 사용하지 않고, 아래 방향을 별도로 재설계합니다.

```text
- Calico 유지 + Cilium chaining + Hubble
- Cilium Service Mesh는 Cilium primary CNI가 가능한 별도 환경에서만 재검증
```
