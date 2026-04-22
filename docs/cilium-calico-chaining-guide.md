# Calico 유지 + Cilium Chaining + Hubble 가이드

- 목적: 기존 Calico를 유지한 상태에서 Cilium chaining과 Hubble를 사용해 sidecar-less 네트워크 가시성을 확보하는 절차를 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-04-17

## 언제 이 방식을 쓰나

- CNI 전체 교체 리스크를 줄이고 싶을 때
- 우선 Hubble UI/CLI와 flow 가시성만 빠르게 보고 싶을 때
- north-south 진입점 교체보다 east-west 관측이 먼저일 때

## 이 방식의 위치

- 현재 프로젝트 우선순위 기준 3순위입니다.
- 이유는 sidecar-less 관측 목적에는 잘 맞지만, `Cilium full replacement`가 아니므로 장기 목표와는 한 단계 거리가 있기 때문입니다.

## 핵심 특징

- Calico가 기본 네트워킹과 IPAM을 계속 담당합니다.
- Cilium은 eBPF 기반 가시성과 일부 기능을 덧붙이는 방식으로 동작합니다.
- Hubble 관측을 빠르게 붙이기 좋습니다.

## 주의점

- Cilium 공식 문서는 chaining 시 일부 고급 기능이 제한될 수 있다고 안내합니다.
- 예시로 `Layer 7 Policy`, `IPsec Transparent Encryption` 제한이 있습니다.
- 기존에 실행 중인 Pod에는 새 chaining 설정이 바로 적용되지 않습니다. 기존 Pod는 재시작해야 chaining 구성이 적용됩니다.
- 이 방식은 `Cilium Ingress`나 `Gateway API`를 장기 표준으로 가져가는 구조와는 목적이 다릅니다.

## 전제 조건

- Calico가 정상 동작 중이어야 합니다.
- 현재 Calico MTU를 확인해야 합니다.
- `portmap` 플러그인 사용을 전제로 합니다.

## 1. 현재 Calico 설정 확인

```bash
kubectl -n kube-system get cm calico-config -o yaml
kubectl -n kube-system get pods -A -o wide
kubectl get no -o wide
```

특히 아래를 확인합니다.

- MTU
- calico kubeconfig 경로
- 현재 Pod 네트워크 정상 여부

## 2. chaining ConfigMap 작성

공식 문서 예시 기준 기본 골격입니다.

```yaml
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
```

적용:

```bash
kubectl apply -f chaining.yaml
```

주의:

- `mtu: 1440`은 공식 예시값입니다.
- 현재 NKS Calico 설정과 다르면 실제 값에 맞춰 조정해야 합니다.

## 3. Cilium 설치

```bash
helm upgrade --install cilium oci://quay.io/cilium/charts/cilium \
  --version 1.19.3 \
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

## 4. 설치 확인

```bash
cilium status --wait
kubectl -n kube-system get pods
kubectl -n kube-system get ds
kubectl -n kube-system get deploy
```

## 5. 기존 Pod 영향

공식 문서 기준 핵심 주의사항입니다.

- 이미 떠 있는 Pod는 새 chaining 설정을 자동으로 쓰지 않습니다.
- 기존 Pod는 reachability는 유지될 수 있지만, 정책 적용과 일부 로드밸런싱이 완전하지 않을 수 있습니다.
- chaining 설정을 실제로 태우려면 해당 Pod들을 재시작해야 합니다.

## 6. Hubble CLI 확인

```bash
cilium hubble port-forward
hubble status
hubble observe
```

필터 예시:

```bash
hubble observe --namespace kube-system
hubble observe --verdict DROPPED
hubble observe --protocol http
```

## 7. Hubble UI 확인

```bash
cilium hubble ui
```

브라우저가 자동으로 안 열리면:

```text
http://localhost:12000
```

## 이 방식의 장점

- full replacement보다 리스크가 낮습니다.
- Hubble 가시성을 먼저 빠르게 검증할 수 있습니다.
- Calico를 유지하므로 초기 롤백 부담이 작습니다.

## 이 방식의 한계

- Cilium의 고급 기능 일부가 제한됩니다.
- 장기 아키텍처 기준선으로 보기 어렵습니다.
- Ingress/Gateway API 중심 구조 검증에는 맞지 않습니다.

## 추천 사용 시점

- Hubble UI/CLI를 가장 낮은 위험으로 먼저 보고 싶을 때
- full replacement 전 사전 검증 단계

## 참고 출처

- https://docs.cilium.io/en/stable/installation/cni-chaining-calico/
- https://docs.cilium.io/en/stable/observability/hubble/setup/
