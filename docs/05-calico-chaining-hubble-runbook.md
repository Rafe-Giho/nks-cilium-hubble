# Calico 유지 + Cilium Chaining + Hubble 실행 가이드

- 목적: NKS 기본 Calico VXLAN을 primary CNI로 유지한 상태에서 Cilium generic-veth chaining과 Hubble로 sidecar-less 네트워크 가시성을 검증합니다.
- 상태: validated-basic
- 마지막 갱신: 2026-05-06

## 현재 판단

이 트랙은 full replacement 실패 이후의 Cilium 기반 1순위 검증안입니다.

핵심 방향:

- Calico 제거 없음
- kube-proxy 대체 없음
- NKS 기본 Calico VXLAN 네트워킹 유지
- Cilium은 chaining으로 붙여 eBPF 기반 관측/Hubble 검증
- 기존 Pod는 자동 적용 대상이 아니므로, 새로 생성한 샘플 Pod부터 검증

## 대상 클러스터

| 항목 | 값 |
| --- | --- |
| 클러스터명 | `ta-sgh-vxlan-sidecarless-cls` |
| kubeconfig | `/mnt/c/Users/user/Downloads/ta-sgh-vxlan-sidecarless-cls_kubeconfig.yaml` |
| context | `nks_ta-sgh-vxlan-sidecarless-cls_3103cceb-5512-4261-9d1e-21485d55eb1c` |
| Kubernetes | `v1.33.4` |
| Worker | 2대 |
| OS | `Ubuntu 22.04.5 LTS` |
| Kernel | `5.15.0-164-generic` |
| Runtime | `containerd://1.6.21` |
| Calico | `v3.30.2` |
| Calico backend | `vxlan` |
| Calico datastore | `etcdv3` |
| metrics API | 정상 |

## 실행 터미널

현재 검증은 Ubuntu Linux 또는 WSL bash 기준입니다.

```bash
export KUBECONFIG="/mnt/c/Users/user/Downloads/ta-sgh-vxlan-sidecarless-cls_kubeconfig.yaml"
export CILIUM_VERSION="1.19.3"
```

## 중단 조건

아래 조건이면 설치하지 않습니다.

- `kubectl get no`에서 Ready가 아닌 노드 존재
- `metrics.k8s.io` APIService가 이미 비정상
- Calico Pod 비정상
- `/etc/cni/net.d/calico/10-calico.conflist` 구조가 이 문서와 다름
- 기존 업무 Pod 재시작 영향 수용 불가
- Helm release 또는 Cilium CRD가 이미 남아 있음

## 0. Preflight

```bash
kubectl config current-context
kubectl get no -o wide
kubectl get pod -A -o wide
kubectl -n kube-system get ds,deploy -o wide
kubectl -n kube-system get cm calico-config -o yaml
kubectl get networkpolicy -A
kubectl get apiservice v1beta1.metrics.k8s.io -o wide
kubectl get --raw /apis/metrics.k8s.io/v1beta1
helm list -A
kubectl get crd
```

확인된 기대 상태:

- 노드 2대 Ready
- `calico-node` 2/2 Ready
- `calico-kube-controllers`, `calico-typha`, `coredns`, `metrics-server` Ready
- NetworkPolicy 없음
- `v1beta1.metrics.k8s.io` Available True
- Helm release 없음
- CRD 없음

## 1. CNI 설정 확인

새 클러스터의 실제 Calico CNI는 Kubernetes datastore 방식이 아니라 etcd 기반입니다.

확인 명령:

```bash
CALICO_POD="$(kubectl -n kube-system get pod -l k8s-app=calico-node -o jsonpath='{.items[0].metadata.name}')"

kubectl -n kube-system exec "$CALICO_POD" -c calico-node -- ls -la /host/etc/cni/net.d
kubectl -n kube-system exec "$CALICO_POD" -c calico-node -- cat /host/etc/cni/net.d/10-calico.conflist
kubectl -n kube-system exec "$CALICO_POD" -c calico-node -- ip -d link show type veth
kubectl -n kube-system exec "$CALICO_POD" -c calico-node -- ip -d link show vxlan.calico
```

확인된 실제 CNI 값:

```json
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "log_file_path": "/var/log/calico/cni/cni.log",
      "etcd_endpoints": "https://192.168.200.82:2379,https://192.168.200.46:2379,https://192.168.200.95:2379",
      "etcd_key_file": "/etc/cni/net.d/calico/calico-tls/etcd-key",
      "etcd_cert_file": "/etc/cni/net.d/calico/calico-tls/etcd-cert",
      "etcd_ca_cert_file": "/etc/cni/net.d/calico/calico-tls/etcd-ca",
      "mtu": 0,
      "ipam": {
        "type": "calico-ipam"
      },
      "policy": {
        "type": "k8s"
      },
      "kubernetes": {
        "kubeconfig": "/etc/cni/net.d/calico/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {"portMappings": true}
    },
    {
      "type": "bandwidth",
      "capabilities": {"bandwidth": true}
    }
  ]
}
```

generic-veth 조건:

- 노드에 `cali...` veth 디바이스 존재 확인
- `vxlan.calico`는 Calico overlay로 유지

## 2. chaining ConfigMap 적용

적용 파일:

- `manifests/05-cni-configuration.yaml`

핵심 차이:

- 공식 예시의 `datastore_type: kubernetes` 사용 안 함
- NKS 실제 Calico etcd endpoint와 인증서 경로 유지
- 기존 `portmap`, `bandwidth` 유지
- 마지막 plugin으로 `cilium-cni` 추가

적용:

```bash
kubectl apply -f manifests/05-cni-configuration.yaml
kubectl -n kube-system get cm cni-configuration -o yaml
```

## 3. Cilium chaining 설치

적용 파일:

- `manifests/05-cilium-chaining-values.yaml`

중요 설정:

```yaml
cluster:
  name: ta-sgh-vxlan-sidecarless-cls
cni:
  chainingMode: generic-veth
  customConf: true
  configMap: cni-configuration
  confPath: /etc/cni/net.d/calico
  exclusive: false
routingMode: native
kubeProxyReplacement: "false"
enableIPv4Masquerade: false
enableIdentityMark: false
hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
```

`cni.confPath=/etc/cni/net.d/calico`가 필요합니다. NKS Calico DaemonSet의 `CNI_NET_DIR`가 해당 경로이기 때문입니다.

설치:

```bash
helm upgrade --install cilium oci://quay.io/cilium/charts/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --values manifests/05-cilium-chaining-values.yaml
```

확인:

```bash
kubectl -n kube-system rollout status ds/cilium --timeout=10m
kubectl -n kube-system rollout status deploy/cilium-operator --timeout=10m
kubectl -n kube-system rollout status deploy/hubble-relay --timeout=10m
kubectl -n kube-system rollout status deploy/hubble-ui --timeout=10m

cilium status --wait
kubectl -n kube-system get pod -l k8s-app=cilium -o wide
kubectl -n kube-system get pod -l k8s-app=hubble-relay -o wide
kubectl -n kube-system get pod -l k8s-app=hubble-ui -o wide
kubectl -n kube-system get svc | grep hubble
```

## 4. 신규 Pod로 chaining 적용 확인

기존 Pod는 Cilium chaining 설정이 자동 적용되지 않습니다. 먼저 새 namespace와 새 Pod만으로 검증합니다.

```bash
kubectl create namespace cilium-chain-check --dry-run=client -o yaml | kubectl apply -f -

kubectl -n cilium-chain-check run curl \
  --image=curlimages/curl:8.10.1 \
  --restart=Never \
  -- sleep 3600

kubectl -n cilium-chain-check wait --for=condition=Ready pod/curl --timeout=3m
kubectl -n cilium-chain-check get pod curl -o wide
```

기본 통신 확인:

```bash
kubectl -n cilium-chain-check exec curl -- curl -sS -k -m 5 https://kubernetes.default.svc/healthz
kubectl -n cilium-chain-check exec curl -- curl -sS -m 5 http://kube-dns.kube-system.svc.cluster.local:9153/metrics
kubectl get apiservice v1beta1.metrics.k8s.io -o wide
kubectl get --raw /apis/metrics.k8s.io/v1beta1
```

기대 결과:

- `curl` Pod가 `Running`
- Pod IP가 Calico IPAM 대역 `10.100.x.x`
- `https://kubernetes.default.svc/healthz` 호출 결과가 `401 Unauthorized`여도 정상
- CoreDNS `:9153/metrics`에서 Prometheus metrics 출력
- `v1beta1.metrics.k8s.io`가 `AVAILABLE=True`
- `/apis/metrics.k8s.io/v1beta1`에서 `APIResourceList` 출력

`401 Unauthorized`는 인증 없이 API server에 접근했기 때문에 발생하는 정상 응답입니다. DNS 해석과 Service 연결 자체가 되는지 확인하는 용도입니다.

## 5. Hubble 확인

`cilium hubble port-forward`가 아래 오류로 끊길 수 있습니다.

```bash
error forwarding port 4245 to pod ...
read: connection reset by peer
error: lost connection to pod
```

이 경우 Hubble Relay 장애로 바로 판단하지 않고, `kubectl port-forward`로 직접 확인합니다.

터미널 1:

```bash
kubectl -n kube-system port-forward svc/hubble-relay 4245:80 --address 127.0.0.1
```

터미널 2:

```bash
hubble status --server localhost:4245
hubble observe --server localhost:4245 --namespace cilium-chain-check --last 40
```

트래픽 생성 후 재확인:

```bash
kubectl -n cilium-chain-check exec curl -- curl -sS -m 5 http://kube-dns.kube-system.svc.cluster.local:9153/metrics >/dev/null
kubectl -n cilium-chain-check exec curl -- curl -sS -k -m 5 https://kubernetes.default.svc/healthz || true

hubble observe --server localhost:4245 --namespace cilium-chain-check --last 40
```

기대 결과:

- `Healthcheck (via localhost:4245): Ok`
- `Connected Nodes: 2/2`
- `cilium-chain-check/curl -> kube-system/coredns:53` flow 확인
- `cilium-chain-check/curl -> kube-system/coredns:9153` flow 확인
- verdict가 `FORWARDED`

아래 경고는 차단 조건이 아닙니다. 장기 검증 전에는 Hubble CLI 버전을 Relay와 맞추는 것을 권장합니다.

```text
Hubble CLI version is lower than Hubble Relay
hubble-cli-version=1.18.6
hubble-relay-version=1.19.3
```

아래 flow도 차단 조건이 아닙니다. IPv6가 비활성인 환경에서 발생하는 Router Solicitation drop입니다.

```text
Unsupported L3 protocol DROPPED (ICMPv6 RouterSolicitation)
```

`--follow`는 port-forward 연결이 유지되는지 볼 때만 사용합니다.

```bash
hubble observe --server localhost:4245 --namespace cilium-chain-check --follow
```

`rpc error: code = Unavailable`, `EOF`, `connect: connection refused`가 나오면 port-forward 세션이 종료된 상태입니다. 터미널 1에서 port-forward를 다시 실행한 뒤 `--last` 조회부터 재확인합니다.

UI 확인:

```bash
cilium hubble ui
```

UI도 동일하게 port-forward가 끊기면 재실행합니다. 운영형 metrics, Grafana, exporter 검토는 설치 성공 후 `docs/06-hubble-observability-runbook.md`에서 진행합니다.

## 5-1. 현재 검증 결과

2026-05-06 사용자 실행 기준:

- `curl` Pod Ready
- `curl` Pod IP: `10.100.188.10`
- API server healthz: `401 Unauthorized`, 네트워크 연결 정상으로 판단
- CoreDNS metrics 접근 성공
- `v1beta1.metrics.k8s.io`: `AVAILABLE=True`
- Hubble Relay healthcheck: `Ok`
- Hubble connected nodes: `2/2`
- `cilium-chain-check/curl`의 DNS/CoreDNS metrics flow 확인
- `cilium hubble port-forward`와 `kubectl port-forward` 모두 장시간 연결 중 reset 가능성 확인
- Hubble CLI `1.18.6`과 Relay `1.19.3` 버전 불일치 경고 확인

## 6. 기존 Pod 재시작 범위

초기 검증 단계에서는 kube-system 전체 재시작 금지.

진행 순서:

1. `cilium-chain-check` 신규 Pod에서 Hubble flow 확인
2. CoreDNS/metrics-server 정상 여부 확인
3. 필요 시 샘플 애플리케이션만 추가 생성
4. 기존 kube-system Pod 재시작은 별도 판단

공식 문서 기준 기존 Pod에는 새 chaining 설정이 자동 적용되지 않습니다. 기존 Pod에서 Cilium 정책/관측을 정확히 보려면 해당 Pod 재시작 필요.

## 7. 롤백

샘플 리소스 제거:

```bash
kubectl delete namespace cilium-chain-check --ignore-not-found
```

Cilium 제거:

```bash
helm -n kube-system uninstall cilium
kubectl -n kube-system delete cm cni-configuration --ignore-not-found
```

CNI 파일이 남아 신규 Pod 생성에 문제가 있으면 노드별로 Cilium CNI 파일을 제거합니다.

```bash
CALICO_PODS="$(kubectl -n kube-system get pod -l k8s-app=calico-node -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}')"

for POD in $CALICO_PODS; do
  kubectl -n kube-system exec "$POD" -c calico-node -- rm -f /host/etc/cni/net.d/05-cilium.conflist
done
```

확인:

```bash
kubectl get no -o wide
kubectl get pod -A -o wide
kubectl get apiservice v1beta1.metrics.k8s.io -o wide
kubectl get --raw /apis/metrics.k8s.io/v1beta1
```

## 성공 기준

- Calico 기존 네트워킹 유지
- Cilium agent 정상
- Hubble Relay/UI 정상
- 새로 생성한 Pod의 흐름이 Hubble에서 보임
- metrics APIService 유지
- `calico-node`를 중단하지 않아도 동작

## 한계

- Cilium full replacement가 아님
- Cilium Ingress/Gateway API 검증 목적이 아님
- L7 Policy, IPsec Transparent Encryption 등 일부 Cilium 기능 제한 가능
- 기존 Pod는 재시작 전까지 Cilium chaining 적용 대상이 아님
- NKS 관리형 Calico 설정 변경 시 `05-cni-configuration.yaml` 재검토 필요

## 참고 출처

- https://docs.cilium.io/en/stable/installation/cni-chaining-calico/
- https://docs.cilium.io/en/stable/installation/cni-chaining-generic-veth.html
- https://docs.cilium.io/en/stable/observability/hubble/setup/
