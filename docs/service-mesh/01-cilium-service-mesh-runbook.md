# Cilium Primary 기반 Service Mesh 실행 가이드

- 목적: Cilium을 primary CNI/kube-proxy replacement로 설치하면서 Gateway API/GAMMA 기반 Cilium Service Mesh를 구성하는 표준 절차를 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-05-14

## 적용 범위

이 문서는 **Cilium이 primary datapath를 담당하는 클러스터**를 대상으로 합니다.
현재 NKS `Calico primary + Cilium generic-veth chaining` 클러스터에 그대로 적용하는 절차가 아닙니다.

현재 NKS chaining 환경에서 Cilium Service Mesh가 실패한 결과는 `docs/service-mesh/05-build-result-report.md`에 정리합니다.

## 핵심 결론

Cilium Service Mesh를 구성하려면 아래 세 가지가 같이 성립해야 합니다.

```text
Cilium primary CNI
  + Cilium kube-proxy replacement
  + Cilium Envoy L7 proxy
```

이 가이드는 운영 검증을 위해 Hubble 관측 계층도 함께 활성화합니다.
다만 Hubble은 Service Mesh datapath 자체가 아니라, Envoy/L7 경로와 네트워크 flow를 확인하기 위한 관측 계층입니다.

역할은 아래처럼 나뉩니다.

| 구성요소 | 역할 |
| --- | --- |
| Cilium CNI/eBPF datapath | Pod datapath, Service datapath, 정책, 로드밸런싱 |
| `kubeProxyReplacement=true` | kube-proxy가 하던 ClusterIP/NodePort/LoadBalancer Service 처리를 Cilium eBPF로 대체 |
| Cilium Envoy | HTTP/gRPC L7 라우팅, Gateway API, GAMMA, CiliumEnvoyConfig 처리 |
| Hubble | Cilium flow, drop, DNS, HTTP L7 metrics/flow 관측 |

Envoy가 kube-proxy를 대체하는 것이 아닙니다.
Cilium eBPF가 kube-proxy를 대체하고, Envoy는 L7 처리를 담당합니다.

## 구축 완료 기준

Helm 설치만으로 Service Mesh가 완료된 것은 아닙니다.

| 단계 | 의미 | 완료 판정 |
| --- | --- | --- |
| Gateway API CRD 설치 | Kubernetes API 타입 추가 | 설치 준비 |
| Cilium 설치 | primary CNI, KPR, Envoy 기능 활성화 | 기능 활성화 |
| Hubble 활성화 | Relay/UI/metrics/exporter 관측 계층 활성화 | 관측 경로 완료 |
| Gateway API 샘플 성공 | 외부 north-south L7 라우팅 검증 | Gateway 경로 완료 |
| GAMMA 샘플 성공 | 내부 east-west Service L7 라우팅 검증 | Mesh 경로 완료 |

## 전제 조건

- 신규 클러스터 또는 Cilium primary CNI 전환이 가능한 클러스터
- kube-proxy 제거 또는 비활성화 가능
- Cilium이 Pod datapath와 Service datapath를 소유할 수 있는 환경
- Gateway API CRD 설치 가능
- LoadBalancer 또는 hostNetwork/NodePort 노출 정책 확정
- Hubble Relay/UI와 Prometheus/Grafana 등 Cilium/Envoy/Hubble metrics 수집 체계 사용 가능

NKS 기본 Calico 클러스터처럼 provider가 primary CNI를 고정하는 환경에서는 먼저 Cilium primary CNI 지원 여부를 공급자에게 확인해야 합니다.

## 변수

```bash
export CILIUM_VERSION="1.19.3"
export GATEWAY_API_VERSION="v1.4.1"
export CLUSTER_NAME="[입력 필요]"
```

## 0. Preflight

```bash
kubectl config current-context
kubectl get no -o wide
kubectl -n kube-system get ds,deploy,pod -o wide
kubectl -n kube-system get ds kube-proxy -o wide || true
kubectl get crd | grep gateway.networking.k8s.io || true
```

확인 기준:

- Cilium을 primary CNI로 설치할 수 있는 클러스터인지 확인
- kube-proxy가 설치돼 있다면 제거 또는 비활성화 계획 확인
- 다른 primary CNI가 이미 Pod datapath를 고정하고 있다면 이 가이드를 바로 적용하지 않음

## 1. Gateway API CRD 설치

Cilium Gateway API/GAMMA는 Gateway API CRD가 필요합니다.

```bash
kubectl apply --server-side -f "https://github.com/kubernetes-sigs/gateway-api/releases/download/${GATEWAY_API_VERSION}/standard-install.yaml"
```

확인:

```bash
kubectl get crd \
  gatewayclasses.gateway.networking.k8s.io \
  gateways.gateway.networking.k8s.io \
  httproutes.gateway.networking.k8s.io \
  grpcroutes.gateway.networking.k8s.io \
  referencegrants.gateway.networking.k8s.io
```

TLSRoute가 필요할 때만 experimental bundle을 추가합니다.

```bash
kubectl apply --server-side -f "https://github.com/kubernetes-sigs/gateway-api/releases/download/${GATEWAY_API_VERSION}/experimental-install.yaml"
```

## 2. Cilium primary values 작성

저장소 샘플:

- `manifests/service-mesh/30-cilium-primary-service-mesh-values.yaml`

핵심 설정:

```yaml
kubeProxyReplacement: "true"
l7Proxy: true

envoyConfig:
  enabled: true

gatewayAPI:
  enabled: true

hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
```

환경에 따라 추가 검토가 필요한 항목:

| 항목 | 검토 기준 |
| --- | --- |
| `routingMode` | cloud/provider 라우팅 방식에 맞춤 |
| `ipam.mode` | Kubernetes host-scope, cluster-pool, cloud ENI 등 환경 기준 |
| `loadBalancer.l7.backend` | CiliumEnvoyConfig L7 LB를 명시적으로 Envoy로 보낼 때 사용 |
| `ingressController.enabled` | Ingress API까지 Cilium으로 처리할 때만 활성화 |
| `gatewayAPI.hostNetwork.enabled` | LoadBalancer 없이 node listener로 노출할 때만 사용 |
| `hubble.metrics.enabled` | HTTP L7, DNS, drop, flow metrics 수집 범위 기준 |
| `hubble.export` | flow 로그 장기 보관이 필요할 때 사용 |

## 3. Helm dry-run

```bash
mkdir -p artifacts/service-mesh

helm upgrade --install cilium oci://quay.io/cilium/charts/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --create-namespace \
  --values manifests/service-mesh/30-cilium-primary-service-mesh-values.yaml \
  --dry-run=client \
  --hide-secret \
  > artifacts/service-mesh/30-cilium-primary-service-mesh-dry-run.yaml
```

확인:

```bash
grep -E "kube-proxy-replacement|enable-gateway-api|enable-envoy-config|enable-l7-proxy|enable-hubble|hubble" artifacts/service-mesh/30-cilium-primary-service-mesh-dry-run.yaml
```

기대값:

- `kube-proxy-replacement: "true"`
- `enable-gateway-api: "true"`
- `enable-envoy-config: "true"`
- `enable-l7-proxy: "true"`
- Hubble Relay/UI/metrics 관련 리소스 렌더링

## 4. Cilium 설치

신규 클러스터 기준:

```bash
helm upgrade --install cilium oci://quay.io/cilium/charts/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --create-namespace \
  --values manifests/service-mesh/30-cilium-primary-service-mesh-values.yaml
```

상태 확인:

```bash
kubectl -n kube-system rollout status ds/cilium --timeout=10m
kubectl -n kube-system rollout status deployment/cilium-operator --timeout=10m
kubectl -n kube-system rollout status deployment/hubble-relay --timeout=5m
kubectl -n kube-system rollout status deployment/hubble-ui --timeout=5m

cilium status --wait
```

Cilium CLI가 없으면 agent 내부에서 확인할 수 있습니다.

```bash
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg status --verbose
```

확인할 값:

- `KubeProxyReplacement: True`
- `Proxy Status: OK`
- `CNI Chaining` 없음 또는 disabled
- `GatewayClass/cilium` 생성
- `cilium-envoy` Ready
- Hubble Relay/UI Ready

## 5. 기본 datapath 검증

```bash
kubectl create ns cilium-service-mesh-check --dry-run=client -o yaml | kubectl apply -f -

kubectl -n cilium-service-mesh-check run curl \
  --image=curlimages/curl:8.10.1 \
  --restart=Never \
  -- sleep 3600

kubectl -n cilium-service-mesh-check wait --for=condition=Ready pod/curl --timeout=3m
kubectl -n cilium-service-mesh-check exec curl -- curl -sS -m 5 http://kube-dns.kube-system.svc.cluster.local:9153/metrics >/dev/null
kubectl -n cilium-service-mesh-check exec curl -- curl -sS -k -m 5 https://kubernetes.default.svc/healthz || true
```

정상 기준:

- 신규 Pod Ready
- DNS 정상
- ClusterIP Service 정상
- API Service 도달

## 6. Gateway API north-south 검증

```bash
kubectl apply -f manifests/service-mesh/20-demo-gateway-http.yaml
kubectl -n cilium-service-mesh-demo rollout status deploy/web-v1 --timeout=3m
kubectl -n cilium-service-mesh-demo rollout status deploy/web-v2 --timeout=3m
kubectl -n cilium-service-mesh-demo get gateway,httproute,svc,pod -o wide
kubectl -n cilium-service-mesh-demo describe gateway mesh-gateway
kubectl -n cilium-service-mesh-demo describe httproute web-route
```

요청:

```bash
GATEWAY_IP="[입력 필요]"
curl -H "Host: mesh-demo.example" "http://${GATEWAY_IP}/"
```

성공 기준:

- `Gateway` Accepted=True
- `Gateway` Programmed=True
- `HTTPRoute` Accepted=True
- HTTP 응답 성공

## 7. GAMMA east-west 검증

GAMMA는 Service를 parent로 하는 HTTPRoute를 통해 내부 Service 트래픽을 per-node Envoy로 전달합니다.
이 가이드는 Cilium 문서 기준 지원되는 producer route, 즉 parent Service와 같은 namespace의 `parentRef=Service` HTTPRoute 검증을 기본 대상으로 둡니다.
consumer route 지원 여부는 사용하는 Cilium/Gateway API 버전의 GAMMA 문서로 별도 확인합니다.

```bash
kubectl apply -f manifests/service-mesh/21-demo-gamma-http-route.yaml
kubectl -n cilium-service-mesh-demo describe httproute gamma-web-route
```

내부 요청:

```bash
kubectl -n cilium-service-mesh-demo run curl \
  --image=curlimages/curl:8.10.1 \
  --restart=Never \
  -- sleep 3600

kubectl -n cilium-service-mesh-demo wait --for=condition=Ready pod/curl --timeout=3m
kubectl -n cilium-service-mesh-demo exec curl -- curl -sS http://web/
```

성공 기준:

- `HTTPRoute/gamma-web-route` Accepted=True
- `ResolvedRefs=True`
- `http://web/` 요청 성공
- 가중치 설정에 따라 `web-v1`, `web-v2` 응답 분산
- Envoy 또는 Cilium proxy metrics 증가

## 8. HTTP L7 Policy 검증

GAMMA와 별개로 Cilium HTTP L7 proxy가 정상인지 확인합니다.

```bash
kubectl create ns cilium-l7-check --dry-run=client -o yaml | kubectl apply -f -

kubectl -n cilium-l7-check create deploy echo \
  --image=nginx:1.27-alpine

kubectl -n cilium-l7-check expose deploy echo --port=80 --target-port=80

kubectl -n cilium-l7-check run curl \
  --image=curlimages/curl:8.10.1 \
  --labels app=curl \
  --restart=Never \
  -- sleep 3600

kubectl -n cilium-l7-check wait --for=condition=Ready pod/curl --timeout=3m
```

L7 policy:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-root-only
  namespace: cilium-l7-check
spec:
  endpointSelector:
    matchLabels:
      app: curl
  egress:
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: cilium-l7-check
            app: echo
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP
          rules:
            http:
              - method: "GET"
                path: "/$"
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s:k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
```

적용 후:

```bash
kubectl apply -f manifests/service-mesh/31-demo-cilium-l7-policy.yaml
kubectl -n cilium-l7-check exec curl -- curl -sS -m 5 -o /tmp/root.out -w '%{http_code}' http://echo/
kubectl -n cilium-l7-check exec curl -- curl -sS -m 5 -o /tmp/blocked.out -w '%{http_code}' http://echo/blocked || true
```

성공 기준:

- `/` 요청은 허용
- `/blocked` 요청은 L7 policy에 의해 차단
- `cilium-dbg status --verbose`에서 active redirect 증가
- Envoy 또는 Cilium proxy metrics 증가

## 9. Cilium Envoy와 Hubble 관측 확인

기본 상태:

```bash
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg status --verbose
kubectl -n kube-system get ds cilium-envoy -o wide
kubectl -n kube-system get deploy hubble-relay hubble-ui -o wide
```

Envoy 로그:

```bash
kubectl -n kube-system logs ds/cilium-envoy --tail=100
```

Hubble CLI가 준비된 경우:

```bash
cilium hubble status
hubble observe --namespace cilium-service-mesh-demo --last 20
hubble observe --namespace cilium-service-mesh-demo --protocol http --last 20
```

Prometheus/Grafana가 구성된 경우 확인 항목:

- Cilium agent metrics
- Cilium Envoy metrics
- Hubble flow/drop/DNS/HTTP metrics
- L7 redirect/proxy 관련 metric
- Gateway/GAMMA 요청 성공률과 latency

주의:

- 이 가이드는 Cilium Service Mesh와 Hubble 관측 계층을 함께 다룹니다.
- Hubble은 flow/L7 visibility 도구이며, 요청 단위 distributed trace의 대체재는 아닙니다.
- 애플리케이션 trace span, request id, call tree는 OpenTelemetry/Tempo/Jaeger 같은 tracing 도구가 필요합니다.

## 10. 정리

샘플 제거:

```bash
kubectl delete -f manifests/service-mesh/31-demo-cilium-l7-policy.yaml --ignore-not-found
kubectl delete -f manifests/service-mesh/21-demo-gamma-http-route.yaml --ignore-not-found
kubectl delete -f manifests/service-mesh/20-demo-gateway-http.yaml --ignore-not-found
kubectl delete ns cilium-l7-check --ignore-not-found
kubectl delete ns cilium-service-mesh-check --ignore-not-found
kubectl delete ns cilium-service-mesh-demo --ignore-not-found
```

Gateway API CRD는 다른 controller가 사용할 수 있으므로 즉시 삭제하지 않습니다.

## 실패 판정 기준

- `KubeProxyReplacement`가 True가 아님
- `Proxy Status`가 OK가 아님
- Gateway/HTTPRoute는 Accepted지만 실제 HTTP 요청 실패
- GAMMA HTTPRoute는 Accepted지만 parent Service 요청 timeout
- HTTP L7 CiliumNetworkPolicy 적용 시 일반 Service 요청이 timeout
- L7 policy 제거 후 요청이 복구됨

위 양상이 나타나면 Cilium Service Mesh datapath가 성립하지 않은 것으로 판단합니다.

## 참고 출처

- Cilium Service Mesh: https://docs.cilium.io/en/stable/network/servicemesh/
- Cilium Gateway API: https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/
- Cilium GAMMA: https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gamma/
- Cilium L7 Traffic Management: https://docs.cilium.io/en/stable/network/servicemesh/l7-traffic-management/
- Cilium kube-proxy replacement: https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/
