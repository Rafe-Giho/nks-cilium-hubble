# Track C. Calico 유지 + Cilium Chaining + Hubble

- 목적: Calico를 primary CNI로 유지한 상태에서 Cilium chaining과 Hubble로 sidecar-less 네트워크 가시성을 검증합니다.
- 상태: draft
- 마지막 갱신: 2026-04-22

## 이 트랙의 위치

이 트랙은 3순위입니다.

full replacement보다 리스크는 낮지만, Cilium Ingress/Gateway API 기반 구조 검증과는 목적이 다릅니다. Hubble 가시성을 먼저 보고 싶을 때만 선택합니다.

## 중단 조건

현재 NKS Calico는 `calico_backend: vxlan`이고 `v3.30.2`입니다. 다만 ConfigMap에는 etcd endpoint 기반 Calico 설정이 보입니다.

아래 조건을 확인하지 못하면 실행하지 않습니다.

- 실제 노드의 `/etc/cni/net.d` Calico 설정 확인 가능
- Calico kubeconfig 경로 확인 가능
- MTU 확정 가능
- 기존 Pod 재시작 영향을 수용 가능

## 0. Preflight

```bash
kubectl get no -o wide
kubectl get pod -A -o wide
kubectl -n kube-system get cm calico-config -o yaml
kubectl get networkpolicy -A
```

기대 상태:

- Calico Pod 정상
- NetworkPolicy 없음
- 기존 업무 Pod 없음 또는 재시작 가능

## 1. CNI 설정 값 확인

Cilium 공식 Calico chaining 예시는 Kubernetes datastore 기준입니다. 현재 NKS Calico는 ConfigMap에 etcd endpoint가 있으므로 공식 예시를 그대로 쓰면 안 됩니다.

확인할 값:

- 현재 노드의 실제 Calico CNI config
- `kubeconfig` 경로
- `datastore_type`
- `mtu`
- `portmap` 포함 여부

노드 CNI 파일을 확인할 수 없으면 여기서 중단합니다.

## 2. chaining ConfigMap 작성

아래는 공식 예시 골격입니다. NKS 실제 Calico CNI 값에 맞춰 수정한 뒤 사용합니다.

```bash
cat > manifests/30-cni-configuration.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: cni-configuration
  namespace: kube-system
data:
  cni-config: |-
    {
      "name": "generic-veth",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "calico",
          "log_level": "info",
          "datastore_type": "kubernetes",
          "mtu": 1440,
          "ipam": {
            "type": "calico-ipam"
          },
          "policy": {
            "type": "k8s"
          },
          "kubernetes": {
            "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
          }
        },
        {
          "type": "portmap",
          "snat": true,
          "capabilities": {"portMappings": true}
        },
        {
          "type": "cilium-cni"
        }
      ]
    }
EOF
```

수정 후 적용:

```bash
kubectl apply -f manifests/30-cni-configuration.yaml
```

## 3. Cilium chaining 설치

```bash
export CILIUM_VERSION="1.19.3"
```

```bash
helm upgrade --install cilium oci://quay.io/cilium/charts/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --set cni.chainingMode=generic-veth \
  --set cni.customConf=true \
  --set cni.configMap=cni-configuration \
  --set routingMode=native \
  --set enableIPv4Masquerade=false \
  --set enableIdentityMark=false \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true
```

확인:

```bash
cilium status --wait
kubectl -n kube-system get pod -l k8s-app=cilium -o wide
kubectl -n kube-system get svc | grep hubble
```

## 4. 기존 Pod 재시작

공식 문서 기준 기존 Pod에는 새 chaining 설정이 자동 적용되지 않습니다. 샘플 워크로드 기준으로만 검증한다면 새 namespace와 새 Pod를 만들어 확인합니다.

```bash
kubectl create namespace demo-a --dry-run=client -o yaml | kubectl apply -f -
kubectl run -n demo-a curl --image=curlimages/curl:8.10.1 --restart=Never -- sleep 3600
```

## 5. Hubble 확인

```bash
cilium hubble port-forward
```

다른 터미널:

```bash
hubble status
hubble observe --namespace demo-a
hubble observe --namespace kube-system
hubble observe --verdict DROPPED
```

UI:

```bash
cilium hubble ui
```

운영형 metrics, Grafana, exporter 검토가 필요하면 Cilium 설치 후 `docs/15-hubble-network-monitoring.md`를 기준으로 별도 선택합니다.

## 성공 기준

- Calico 기존 네트워킹 유지
- Cilium agent 정상
- Hubble Relay/UI 정상
- 새로 생성한 Pod의 흐름이 Hubble에서 보임

## 한계

- Cilium full replacement가 아닙니다.
- Cilium Ingress/Gateway API 검증에는 적합하지 않습니다.
- L7 Policy, IPsec Transparent Encryption 등 일부 Cilium 기능은 제한될 수 있습니다.
- NKS Calico 설정이 공식 예시와 다를 수 있어 실제 노드 CNI 파일 확인이 필요합니다.

## 참고 출처

- https://docs.cilium.io/en/stable/installation/cni-chaining-calico/
- https://docs.cilium.io/en/stable/observability/hubble/setup/
