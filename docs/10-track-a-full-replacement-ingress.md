# Track A. Cilium Full Replacement + Ingress + Hubble

- 목적: NKS 기본 Calico 클러스터를 Cilium primary CNI로 전환하고, Cilium Ingress와 Hubble Relay/UI/CLI를 검증합니다.
- 상태: draft
- 마지막 갱신: 2026-04-22

## 이 트랙의 위치

이 트랙이 현재 기본안입니다.

구성 목표:

- NKS 기본 Calico VXLAN 기반 CNI에서 Cilium primary CNI로 전환
- 신규 Pod 네트워킹을 Cilium이 담당하도록 CNI 설정 소유권 전환
- Hubble Relay/UI/CLI 활성화
- Cilium Ingress 활성화
- 샘플 HTTP 워크로드로 north-south/east-west 트래픽 확인

## 가능 여부 판단

이 트랙은 `NKS가 공식 제공하는 CNI 변경 기능`을 사용하는 절차가 아닙니다.

현재 확인한 NHN Cloud NKS 문서 기준으로 클러스터 생성 시 선택 가능한 CNI는 Calico 계열이고, release notes에는 cluster CNI change 기능이 더 이상 지원되지 않는다고 명시되어 있습니다. 따라서 이 절차는 공식 지원 구축이 아니라 `NKS 위에서 Cilium migration 절차를 실험하는 unsupported PoC`입니다.

기술적으로는 Cilium 공식 migration 방식처럼 기존 Calico와 다른 새 Pod CIDR, cluster-pool IPAM, Cilium 기본 VXLAN port, 노드별 전환 절차를 사용해 시도할 수 있습니다. 성공 여부는 실제 클러스터에서 아래 항목을 모두 통과해야 판단합니다.

- 모든 기존 worker node에서 Cilium agent가 정상
- 새로 생성되는 Pod IP가 Cilium Pod CIDR 안에 있음
- CoreDNS, Kubernetes Service, kube-system 통신 정상
- Cilium Ingress LoadBalancer와 backend 통신 정상
- Hubble Relay/UI/CLI 정상
- 신규 node scale-out 후에도 Cilium이 CNI 소유권을 가져감

## 목표 상태

최종 목표 상태는 아래와 같습니다.

- Cilium이 primary CNI입니다.
- `cni.customConf=false`, `cni-exclusive=true` 기준으로 Cilium이 CNI config를 씁니다.
- Cilium 정책 enforcement는 `default`입니다.
- Hubble Relay/UI가 활성화되어 있습니다.
- Cilium Ingress가 활성화되어 있습니다.
- Calico는 더 이상 신규 Pod 네트워킹 경로가 아니어야 합니다.
- 신규 노드는 Cilium 준비 전 일반 Pod를 실행하지 않도록 초기 taint를 가집니다.
- Cilium agent는 해당 taint를 toleration하고, 준비 완료 후 taint를 제거합니다.

주의: NKS 관리형 Calico add-on 또는 노드 부트스트랩이 Calico 리소스/CNI 파일을 다시 만들 수 있습니다. 이 가능성이 확인되지 않으면 운영 가능한 full replacement로 판단하지 않습니다.

## 중단 조건

아래 조건 중 하나라도 해결되지 않으면 설치를 시작하지 않습니다.

- NKS에서 CNI 변경이 공식 비지원 PoC라는 점을 수용하지 못함
- 기존 Pod CIDR, Service CIDR, Node/VPC CIDR과 겹치지 않는 새 Cilium Pod CIDR을 확정하지 못함
- 노드 간 Cilium VXLAN `UDP 8472`와 Hubble `TCP 4244` 허용 여부를 확인하지 못함
- 신규 node scale-out 시 `node.cilium.io/agent-not-ready=true:NoExecute` 또는 동등한 초기 taint를 적용할 방법을 확인하지 못함
- Cilium Helm `agentNotReadyTaintKey`와 NKS node group taint key를 일치시키지 못함
- PoC 실패 시 클러스터 재생성 롤백을 수용하지 못함

## 자동화 범위

이 가이드에서 말하는 scale-out 자동화는 `신규 노드 생성 후 Cilium이 먼저 기동되고, CNI 설정을 가져간 뒤, 일반 Pod가 스케줄되는 흐름`입니다.

자동화되는 부분:

- Cilium DaemonSet이 node group 기본 taint를 toleration합니다.
- Cilium agent가 준비되면 `agentNotReadyTaintKey` taint를 제거합니다.
- 최종 values의 `cni.customConf=false` 기준으로 Cilium이 CNI config를 씁니다.
- Hubble Relay/UI와 Cilium Ingress 리소스는 Helm values로 재현 가능합니다.

자동화되지 않거나 사전 설정이 필요한 부분:

- NKS가 공식 CNI 교체를 지원하지 않으므로 NKS add-on 관점의 완전 대체는 자동화할 수 없습니다.
- NKS node group에 신규 노드 기본 taint를 설정하는 작업은 콘솔/API에서 먼저 가능 여부를 확인해야 합니다.
- 기존 노드는 최초 1회 node-by-node migration이 필요합니다.
- NKS가 노드 재부팅, 업데이트, scale-out 시 Calico CNI 파일을 되돌리는지 실제 검증해야 합니다.

따라서 이 트랙의 자동화 성공 판정은 `NKS node group taint 설정 가능`, `Cilium toleration 적용`, `Cilium taint 제거`, `신규 Pod Cilium CIDR 할당`을 모두 통과한 뒤에만 내립니다.

## 0. 실행 변수 지정

아래 값은 WSL 세션에서 지정합니다.

```bash
export CILIUM_VERSION="1.19.3"
export CILIUM_POD_CIDR="<NEW_DISTINCT_CILIUM_POD_CIDR>"
export CILIUM_TUNNEL_PORT="8472"
export DEMO_NAMESPACE="demo-a"
export DEMO_HOST="demo.example.local"
export CILIUM_AGENT_NOT_READY_TAINT_KEY="node.cilium.io/agent-not-ready"
export CILIUM_AGENT_NOT_READY_TAINT_VALUE="true"
export CILIUM_AGENT_NOT_READY_TAINT_EFFECT="NoExecute"
```

CIDR을 확정하지 않았다면 여기서 중단합니다.

```bash
test "$CILIUM_POD_CIDR" != "<NEW_DISTINCT_CILIUM_POD_CIDR>" || {
  echo "STOP: CILIUM_POD_CIDR is not set"
  exit 1
}
```

현재 NKS에서 관측된 Pod IP는 `10.100.184.0/24`, `10.100.188.0/24` 대역이고, Service IP는 `10.254.x.x` 대역입니다. 새 Cilium Pod CIDR은 이 대역들과 겹치면 안 됩니다.

## 0-1. NKS 보안그룹 확인

현재 NKS Calico 기본 보안그룹은 worker security group self 원격으로 `TCP 1-65535`, `UDP 1-65535` 수신을 허용합니다. 이 규칙이 유지된다면 Cilium 기본 VXLAN `UDP 8472`, Hubble server `TCP 4244`, Cilium health `TCP 4240`은 worker node 간 통신 기준으로 이미 포함됩니다.

NKS Calico VXLAN 기본값은 `UDP 4789`이고 Cilium VXLAN 기본값은 `UDP 8472`이므로 기본값끼리 충돌하지 않습니다.

설치 전 아래 문서를 확인합니다.

```text
docs/06-nks-security-group.md
```

PoC 기준 보안그룹 조치:

| 구분 | 조치 |
| --- | --- |
| 추가 | 없음. 현재 worker self `TCP/UDP 1-65535`가 유지되는 경우 |
| 수정 | 없음 |
| 삭제 | 없음. Calico/NKS control plane 관련 규칙은 PoC 중 유지 |

## 1. Preflight 스냅샷

로컬 산출물 저장 디렉터리를 만듭니다.

```bash
mkdir -p artifacts/preflight
```

현재 상태를 저장합니다.

```bash
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

기대 상태:

- 노드 3대 `Ready`
- NetworkPolicy 없음
- Ingress 없음
- LoadBalancer Service 없음
- Calico `calico_backend: vxlan`
- Calico image `v3.30.2`
- 신규 노드에 초기 taint를 넣을 수 있는지 NKS node group 설정 또는 후속 자동화 방식 확인
- NKS 보안그룹에서 worker self `TCP/UDP 1-65535` 또는 Cilium 최소 포트 허용 확인

## 1-1. Node scale-out 자동화 전제 확인

Cilium이 단일 CNI로 동작하려면 신규 노드에서 Cilium agent가 CNI config를 쓰기 전에 일반 Pod가 먼저 뜨면 안 됩니다.

Cilium 공식 문서는 지원되지 않는 단일 CNI 환경에서 cloud provider가 CNI 설정을 되돌리거나, Cilium 시작 전 Pod가 떠서 기존 CNI IP를 받을 수 있다고 설명합니다. 이를 줄이기 위해 신규 노드에 `node.cilium.io/agent-not-ready` taint를 두고, Cilium이 준비되면 taint를 제거하는 방식을 사용합니다.

taint effect는 `NoExecute`를 기본값으로 둡니다. Cilium 공식 문서는 지원되지 않는 단일 CNI 환경에서 이미 스케줄된 Pod가 Cilium보다 먼저 실행되는 문제까지 줄이려면 `NoExecute`가 더 적합하다고 설명합니다.

NKS에서 scale-out을 자동화하려면 아래 흐름이 성립해야 합니다.

1. NKS node group에 신규 노드 기본 taint를 설정합니다.
2. 신규 노드가 생성되면 taint 때문에 일반 workload가 먼저 뜨지 않습니다.
3. Cilium DaemonSet은 해당 taint를 toleration하고 신규 노드에 먼저 뜹니다.
4. Cilium agent가 CNI config를 쓰고 준비되면 taint를 제거합니다.
5. 그 뒤 일반 Pod가 신규 노드에 스케줄되고 Cilium CNI를 사용합니다.

NKS node group taint 설정:

```text
key: node.cilium.io/agent-not-ready
value: true
effect: NoExecute
```

중요:

- 이 설정은 신규 노드에 적용되는 것을 목표로 합니다.
- 기존 노드는 이 taint 자동화 대신 node-by-node migration 절차로 전환합니다.
- NKS node group 수준에서 taint를 신규 노드에 자동 적용할 수 있는지 콘솔에서 확인해야 합니다.
- Autoscaler 사용 시에는 Cilium 문서 기준 `ignore-taint.cluster-autoscaler.kubernetes.io/...` 계열 key가 필요할 수 있습니다. 이 경우 NKS node group taint key와 Cilium Helm `agentNotReadyTaintKey`를 같은 값으로 바꿔야 합니다.

확인할 것:

```bash
kubectl get no -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}'
```

이 값을 node group 수준에서 자동 적용할 수 없으면, scale-out은 자동 안정화된 상태가 아닙니다. 그 경우 신규 노드는 생성 후 수동 검증 절차를 반드시 거쳐야 합니다.

## 1-2. Cilium taint/toleration 자동화 값

NKS node group taint key와 Cilium Helm `agentNotReadyTaintKey`는 반드시 같아야 합니다.

기본값을 사용할 경우:

```yaml
agentNotReadyTaintKey: node.cilium.io/agent-not-ready
tolerations:
  - operator: Exists
```

Autoscaler가 taint를 무시해야 하는 환경이면 예를 들어 아래처럼 바꿉니다.

```bash
export CILIUM_AGENT_NOT_READY_TAINT_KEY="ignore-taint.cluster-autoscaler.kubernetes.io/cilium-agent-not-ready"
```

이 경우 NKS node group taint도 같은 key로 설정합니다.

```text
key: ignore-taint.cluster-autoscaler.kubernetes.io/cilium-agent-not-ready
value: true
effect: NoExecute
```

이 프로젝트의 기본 PoC는 Autoscaler 없이 수동 node scale-out 검증을 먼저 수행하므로 기본 key인 `node.cilium.io/agent-not-ready`를 우선 사용합니다.

## 2. Cilium secondary mode values 작성

이 단계는 Cilium을 바로 primary CNI로 만들지 않고, 별도 overlay를 먼저 구성하는 migration 시작점입니다.

```bash
cat > manifests/10-cilium-values-initial.yaml <<EOF
operator:
  unmanagedPodWatcher:
    restart: false
agentNotReadyTaintKey: ${CILIUM_AGENT_NOT_READY_TAINT_KEY}
tolerations:
  - operator: Exists
routingMode: tunnel
tunnelProtocol: vxlan
tunnelPort: ${CILIUM_TUNNEL_PORT}
cni:
  customConf: true
  uninstall: false
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
      - "${CILIUM_POD_CIDR}"
policyEnforcementMode: never
bpf:
  hostLegacyRouting: true
hubble:
  relay:
    enabled: true
  ui:
    enabled: true
EOF
```

파일을 확인합니다.

```bash
cat manifests/10-cilium-values-initial.yaml
```

## 3. Helm values dry-run 생성

```bash
cilium install \
  --version "${CILIUM_VERSION}" \
  --values manifests/10-cilium-values-initial.yaml \
  --dry-run-helm-values \
  > manifests/10-cilium-values-initial.generated.yaml

cat manifests/10-cilium-values-initial.generated.yaml
```

확인할 값:

- `ipam.mode: cluster-pool`
- `clusterPoolIPv4PodCIDRList`가 새 CIDR
- `tunnelPort: 8472`
- `cni.customConf: true`
- `policyEnforcementMode: never`
- `agentNotReadyTaintKey`가 NKS node group taint key와 같음
- `tolerations`에 Cilium agent가 신규 노드 taint를 toleration할 수 있는 설정이 있음
- `hubble.relay.enabled: true`
- `hubble.ui.enabled: true`

## 4. Cilium secondary mode 설치

이 단계부터 클러스터 변경입니다.

```bash
helm upgrade --install cilium oci://quay.io/cilium/charts/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --values manifests/10-cilium-values-initial.generated.yaml
```

상태 확인:

```bash
cilium status --wait
kubectl -n kube-system get pod -l k8s-app=cilium -o wide
kubectl -n kube-system get deploy,ds,svc | grep -E 'cilium|hubble'
kubectl -n kube-system get ds cilium -o jsonpath='{.spec.template.spec.tolerations}{"\n"}'
```

기대 상태:

- Cilium agent/operator가 정상 기동
- 아직 기존 Pod 대부분은 Calico CNI 소속
- Hubble Relay/UI 리소스 생성
- Cilium DaemonSet toleration에 신규 노드 taint를 허용할 수 있는 설정이 있음

## 5. Per-node migration 설정 생성

초기에는 어떤 노드에도 적용되지 않는 `CiliumNodeConfig`를 생성합니다.

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
    write-cni-conf-when-ready: /host/etc/cni/net.d/05-cilium.conflist
    custom-cni-conf: "false"
    cni-chaining-mode: "none"
    cni-exclusive: "true"
EOF
```

확인:

```bash
kubectl -n kube-system get ciliumnodeconfig
```

## 6. 노드 단위 migration

아래 절차를 노드마다 한 번씩 반복합니다. 현재 노드는 3대입니다.

```bash
kubectl get no -o name
```

첫 번째 노드를 선택합니다.

```bash
export NODE="<NODE_NAME>"
```

노드 cordon/drain:

```bash
kubectl cordon "$NODE"
kubectl drain "$NODE" --ignore-daemonsets --delete-emptydir-data
```

노드에 Cilium CNI 적용 label을 붙입니다.

```bash
kubectl label node "$NODE" --overwrite io.cilium.migration/cilium-default=true
```

해당 노드의 Cilium agent를 재시작해 CNI 설정 파일을 쓰게 합니다.

```bash
kubectl -n kube-system delete pod \
  --field-selector spec.nodeName="$NODE" \
  -l k8s-app=cilium

kubectl -n kube-system rollout status ds/cilium -w
```

노드를 재부팅합니다.

```text
NKS 콘솔 또는 승인된 운영 방식으로 NODE를 재부팅합니다.
```

재부팅 후 검증합니다.

```bash
kubectl wait --for=condition=Ready node/"$NODE" --timeout=10m
cilium status --wait
kubectl get no "$NODE" -o wide
```

해당 노드에 임시 Pod를 띄워 API server 접근을 확인합니다.

```bash
kubectl -n kube-system run verify-network \
  --attach --rm --restart=Never \
  --overrides='{"spec":{"nodeName":"'"$NODE"'","tolerations":[{"operator":"Exists"}]}}' \
  --image ghcr.io/nicolaka/netshoot:v0.13 \
  -- /bin/bash -c 'ip -br addr && curl -s -k https://$KUBERNETES_SERVICE_HOST/healthz && echo'
```

정상이면 노드를 다시 스케줄 가능하게 합니다.

```bash
kubectl uncordon "$NODE"
```

모든 노드에 대해 위 절차를 반복합니다.

## 7. 기존 Calico CNI 경로 배제 확인

모든 노드 전환 후, 새 Pod가 Cilium CIDR을 받는지 먼저 확인합니다.

```bash
kubectl create namespace cilium-check --dry-run=client -o yaml | kubectl apply -f -

kubectl -n cilium-check run ip-check \
  --image=busybox:1.36 \
  --restart=Never \
  -- sleep 3600

kubectl -n cilium-check wait --for=condition=Ready pod/ip-check --timeout=3m
kubectl -n cilium-check get pod ip-check -o wide
```

확인 기준:

- Pod IP가 `CILIUM_POD_CIDR` 안에 있어야 합니다.
- Pod IP가 기존 Calico 관측 대역인 `10.100.184.0/24`, `10.100.188.0/24`에 있으면 전환 실패로 봅니다.

정리:

```bash
kubectl delete namespace cilium-check
```

## 8. 전체 migration 확인

```bash
cilium status --wait
kubectl get no -o wide
kubectl get pod -A -o wide
```

확인 기준:

- 모든 노드가 `Ready`
- Cilium status가 정상
- 새로 뜨는 Pod IP가 `CILIUM_POD_CIDR` 안에 있음
- CoreDNS가 정상 동작

DNS 확인:

```bash
kubectl run dns-check --rm -it --restart=Never \
  --image=busybox:1.36 \
  -- nslookup kubernetes.default.svc.cluster.local
```

## 9. Cilium primary CNI + Ingress 최종 values 적용

최종 상태에서는 Cilium을 primary CNI로 두고, 정책 enforcement와 Ingress를 켭니다.

```bash
cilium install \
  --version "${CILIUM_VERSION}" \
  --values manifests/10-cilium-values-initial.generated.yaml \
  --dry-run-helm-values \
  --set operator.unmanagedPodWatcher.restart=true \
  --set cni.customConf=false \
  --set policyEnforcementMode=default \
  --set bpf.hostLegacyRouting=false \
  --set agentNotReadyTaintKey="${CILIUM_AGENT_NOT_READY_TAINT_KEY}" \
  --set tolerations[0].operator=Exists \
  --set kubeProxyReplacement=true \
  --set ingressController.enabled=true \
  --set ingressController.loadbalancerMode=dedicated \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  > manifests/10-cilium-values-final-ingress.yaml

diff -u manifests/10-cilium-values-initial.generated.yaml manifests/10-cilium-values-final-ingress.yaml || true
```

적용:

```bash
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --values manifests/10-cilium-values-final-ingress.yaml

kubectl -n kube-system rollout restart ds/cilium
kubectl -n kube-system rollout restart deploy/cilium-operator
cilium status --wait
```

최종 CNI 설정 확인:

```bash
kubectl -n kube-system get cm cilium-config -o yaml | grep -E 'custom-cni-conf|cni-exclusive|agent-not-ready-taint-key'
kubectl -n kube-system get ds cilium -o jsonpath='{.spec.template.spec.tolerations}{"\n"}'
```

## 9-1. Per-node migration 설정 정리

이 단계는 모든 기존 노드 migration과 최종 values 적용이 끝난 뒤에만 수행합니다.

Cilium 공식 migration 절차는 post-migration 단계에서 per-node 설정을 삭제합니다. 이 PoC에서도 최종 `cni.customConf=false`가 정상 적용된 뒤에는 신규 노드가 node label 없이도 Cilium global 설정을 따라야 하므로, scale-out 검증 전에 per-node 설정 의존성을 제거합니다.

현재 상태 확인:

```bash
kubectl -n kube-system get ciliumnodeconfig cilium-default
kubectl get no -l io.cilium.migration/cilium-default=true
```

정리:

```bash
kubectl delete -n kube-system ciliumnodeconfig cilium-default
kubectl label node --all io.cilium.migration/cilium-default-
```

확인:

```bash
kubectl -n kube-system get ciliumnodeconfig
kubectl get no -L io.cilium.migration/cilium-default
cilium status --wait
```

기대 상태:

- `CiliumNodeConfig`가 없어도 Cilium이 정상
- 기존 노드의 `io.cilium.migration/cilium-default` label이 제거됨
- 새로 뜨는 Pod가 계속 `CILIUM_POD_CIDR`을 받음

## 10. Scale-out 자동화 검증

이 단계는 “노드 스케일링 발생 시 자동으로 Cilium으로 되는가”를 판단하는 핵심 검증입니다.

NKS 콘솔에서 worker node를 1대 추가합니다. 추가 전에 node group에 아래 taint가 신규 노드 기본값으로 적용되도록 설정합니다.

```text
key: ${CILIUM_AGENT_NOT_READY_TAINT_KEY}
value: true
effect: NoExecute
```

자동화 검증이 끝날 때까지 일반 워크로드를 추가 배포하지 않습니다.

신규 노드 확인:

```bash
kubectl get no -o wide
kubectl get no -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}'
kubectl -n kube-system get pod -l k8s-app=cilium -o wide
kubectl -n kube-system get cm cilium-config -o yaml | grep -E 'agent-not-ready-taint-key|write-cni-conf|custom-cni-conf|cni-exclusive'
kubectl get no -L io.cilium.migration/cilium-default
cilium status --wait
```

신규 노드 이름을 지정합니다.

```bash
export NEW_NODE="<NEW_NODE_NAME>"
```

신규 노드에 검증 Pod를 강제로 올립니다.

```bash
kubectl -n kube-system run verify-new-node \
  --attach --rm --restart=Never \
  --overrides='{"spec":{"nodeName":"'"$NEW_NODE"'","tolerations":[{"operator":"Exists"}]}}' \
  --image ghcr.io/nicolaka/netshoot:v0.13 \
  -- /bin/bash -c 'ip -br addr && curl -s -k https://$KUBERNETES_SERVICE_HOST/healthz && echo'
```

판단 기준:

- 신규 노드에 Cilium agent가 떠야 합니다.
- Cilium agent가 준비되기 전에는 신규 노드에 agent-not-ready taint가 남아 있어야 합니다.
- Cilium agent가 준비된 뒤에는 해당 taint가 제거되어야 합니다.
- 검증 Pod IP가 `CILIUM_POD_CIDR` 안에 있어야 합니다.
- Cilium이 준비되기 전 일반 Pod가 신규 노드에 뜨면 안 됩니다.
- 신규 노드에 `io.cilium.migration/cilium-default` label을 수동으로 붙이지 않아도 Cilium CNI가 적용되어야 합니다.
- 신규 노드에서 CNI config가 Calico로 되돌아가면 full replacement 자동화 실패입니다.

taint 제거 확인:

```bash
kubectl get node "$NEW_NODE" -o jsonpath='{.spec.taints}{"\n"}'
```

위 출력에 `${CILIUM_AGENT_NOT_READY_TAINT_KEY}`가 남아 있으면 Cilium이 신규 노드를 아직 준비 완료로 처리하지 못한 것입니다.

결론:

- 위 검증을 통과하면 NKS scale-out 후에도 Cilium 적용 가능성이 있습니다.
- node group taint, Cilium toleration, Cilium taint 제거가 모두 동작하면 신규 노드 scale-out 자동화는 PoC 기준 통과입니다.
- node group 수준 taint 또는 동등 자동화 없이 통과하지 못하면, scale-out은 수동 검증/보정 대상입니다.
- 이 검증 전에는 “노드 스케일링도 자동으로 Cilium이 된다”고 판단하지 않습니다.

## 11. Ingress 샘플 워크로드 배포

```bash
kubectl create namespace "${DEMO_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
```

샘플 앱:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-echo
  namespace: ${DEMO_NAMESPACE}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-echo
  template:
    metadata:
      labels:
        app: demo-echo
    spec:
      containers:
        - name: agnhost
          image: registry.k8s.io/e2e-test-images/agnhost:2.53
          args: ["netexec", "--http-port=8080"]
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: demo-echo
  namespace: ${DEMO_NAMESPACE}
spec:
  selector:
    app: demo-echo
  ports:
    - name: http
      port: 80
      targetPort: 8080
EOF
```

Ingress:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  namespace: ${DEMO_NAMESPACE}
  annotations:
    ingress.cilium.io/loadbalancer-mode: dedicated
spec:
  ingressClassName: cilium
  rules:
    - host: ${DEMO_HOST}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: demo-echo
                port:
                  number: 80
EOF
```

확인:

```bash
kubectl -n "${DEMO_NAMESPACE}" get deploy,svc,ingress -o wide
kubectl get svc -A | grep -E 'cilium|demo|LoadBalancer'
```

LoadBalancer IP 또는 hostname이 생길 때까지 기다립니다.

```bash
kubectl -n "${DEMO_NAMESPACE}" get ingress demo-ingress -w
```

호출:

```bash
export INGRESS_ADDR="$(kubectl -n "${DEMO_NAMESPACE}" get ingress demo-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
if [ -z "$INGRESS_ADDR" ]; then
  export INGRESS_ADDR="$(kubectl -n "${DEMO_NAMESPACE}" get ingress demo-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
fi

curl -v -H "Host: ${DEMO_HOST}" "http://${INGRESS_ADDR}/"
```

## 12. Hubble CLI/UI 확인

Relay port-forward:

```bash
cilium hubble port-forward
```

다른 터미널에서:

```bash
hubble status
hubble observe --namespace "${DEMO_NAMESPACE}"
hubble observe --namespace "${DEMO_NAMESPACE}" --protocol http
hubble observe --verdict DROPPED
```

UI:

```bash
cilium hubble ui
```

브라우저가 자동으로 열리지 않으면:

```text
http://localhost:12000
```

UI에서 확인할 것:

- `demo-a` namespace의 Pod 간 트래픽
- Ingress Envoy에서 backend Pod로 가는 트래픽
- DNS 질의
- dropped flow 여부

Grafana metrics, 내부 Ingress 노출, exporter 기반 장기 보관은 `docs/15-hubble-network-monitoring.md`를 기준으로 별도 선택합니다.

## 13. 최종 성공 기준과 중단 기준

PoC 성공 기준:

- 모든 노드가 Cilium primary CNI로 동작
- 신규 scale-out 노드도 Cilium primary CNI로 동작
- Hubble Relay/UI/CLI 정상
- Cilium Ingress가 LoadBalancer를 받고 샘플 앱 호출 성공
- Hubble에서 HTTP/DNS 흐름 확인

중단 기준:

- 노드가 `Ready`로 복귀하지 않음
- CoreDNS 장애
- API server 접근 실패
- Cilium status 비정상
- Ingress LoadBalancer 생성 실패가 장시간 지속
- 신규 노드가 Calico CNI config 또는 Calico Pod IP 대역을 계속 사용
- Cilium 준비 전 일반 Pod가 신규 노드에 스케줄됨

## 14. Calico 리소스 처리 원칙

이 트랙의 전제는 “신규 Pod 네트워킹을 Cilium이 담당한다”입니다. 따라서 최종 성공 판단에서 중요한 것은 Calico 리소스 존재 여부가 아니라, 실제 CNI config와 신규 Pod IPAM이 Cilium으로 전환됐는지입니다.

Calico 삭제는 별도 판단이 필요합니다.

- NKS 관리형 add-on이 Calico를 재생성할 수 있습니다.
- Calico 삭제가 NKS add-on 관리 상태와 충돌할 수 있습니다.
- Calico 삭제 후 노드 재부팅/스케일링 시 부트스트랩이 어떤 CNI 파일을 다시 쓰는지 확인해야 합니다.

따라서 이 가이드에서는 Calico 삭제 명령을 기본 절차에 넣지 않습니다. 대신 신규 Pod와 신규 노드가 Cilium CIDR/CNI를 사용하는지로 full replacement 여부를 판단합니다.

## 참고 출처

- https://docs.cilium.io/en/stable/installation/k8s-install-migration/
- https://docs.cilium.io/en/stable/installation/k8s-install-helm/
- https://docs.cilium.io/en/stable/network/servicemesh/ingress/
- https://docs.cilium.io/en/stable/observability/hubble/setup/
