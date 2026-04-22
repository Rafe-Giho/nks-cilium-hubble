# Cilium Gateway API 가이드

- 목적: Cilium이 설치된 클러스터에서 Gateway API를 활성화하고 검증하는 절차를 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-04-17

## 언제 이 방식을 쓰나

- 장기적으로 Ingress보다 구조적인 API를 원할 때
- HTTP 외에 gRPC나 TLSRoute까지 고려할 때
- 라우팅과 진입점 구성을 역할별로 나누고 싶을 때
- 향후 GAMMA 또는 서비스메쉬 확장까지 고려할 때

## 전제 조건

- Cilium이 먼저 설치되어 있어야 합니다.
- `kubeProxyReplacement=true`가 필요합니다.
- `l7Proxy=true`가 필요합니다.
- Gateway API CRD를 미리 설치해야 합니다.
- 기본 노출 방식은 `LoadBalancer`입니다.

## 1. Gateway API CRD 설치

Cilium stable 1.19.3 문서 기준 Gateway API `v1.4.1` CRD를 먼저 설치합니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_gateways.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_grpcroutes.yaml
```

TLSRoute까지 볼 계획이면 아래도 추가합니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml
```

## 2. Gateway API 활성화

```bash
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --version 1.19.3 \
  --namespace kube-system \
  --reuse-values \
  --set kubeProxyReplacement=true \
  --set gatewayAPI.enabled=true

kubectl -n kube-system rollout restart deployment/cilium-operator
kubectl -n kube-system rollout restart ds/cilium
```

## 3. 기본 확인

```bash
cilium status
kubectl get gatewayclass
kubectl get gateway -A
kubectl get httproute -A
```

## 4. 최소 리소스 예시

아래는 HTTPRoute 기반 최소 예시입니다.

### Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: demo-gateway
  namespace: demo-a
spec:
  gatewayClassName: cilium
  listeners:
    - name: web
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
```

### HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-route
  namespace: demo-a
spec:
  parentRefs:
    - name: demo-gateway
  hostnames:
    - demo.example.local
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: demo-echo
          port: 80
```

## 5. 확인 포인트

```bash
kubectl get gateway -A
kubectl describe gateway -n demo-a demo-gateway
kubectl get httproute -A
kubectl describe httproute -n demo-a demo-route
kubectl get svc -A
```

중요하게 볼 상태는 아래입니다.

- Gateway `Accepted=True`
- Gateway `Programmed=True`
- Listener `Programmed=True`
- Route가 정상 attach 되었는지

## Hubble 관점 확인

```bash
cilium hubble port-forward
hubble status
hubble observe --namespace demo-a
```

Hubble UI에서는 아래를 봅니다.

- 외부 요청이 Gateway listener로 들어오는지
- Gateway에서 backend로 연결되는지
- route attach 후 트래픽이 정상 전달되는지

## Hubble UI 접속

```bash
cilium hubble ui
```

## hostNetwork 대안

LoadBalancer 없이 외부 LB와 결합하려면 Gateway API host network 모드를 사용할 수 있습니다.

```yaml
gatewayAPI:
  enabled: true
  hostNetwork:
    enabled: true
```

노드 일부에만 listener를 올리는 것도 가능합니다.

```yaml
gatewayAPI:
  enabled: true
  hostNetwork:
    enabled: true
    nodes:
      matchLabels:
        role: infra
```

## 이 방식의 장점

- Ingress보다 구조가 명확합니다.
- `Gateway`와 `Route`를 분리해 운영하기 좋습니다.
- HTTPRoute, GRPCRoute, TLSRoute 등 확장성이 좋습니다.
- 장기적으로는 이쪽이 더 표준적인 방향입니다.

## 이 방식의 한계

- CRD 설치가 필요합니다.
- 리소스 수와 개념 수가 늘어납니다.
- 첫 PoC 기준으로는 Ingress보다 진입 장벽이 높습니다.

## 추천 사용 시점

- 1차 PoC 이후 구조 확장 검토
- 장기 표준 API 검토
- 멀티팀 운영 모델과 고급 라우팅이 중요할 때

## 참고 출처

- https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/
- https://gateway-api.sigs.k8s.io/concepts/api-overview/
- https://gateway-api.sigs.k8s.io/guides/getting-started/migrating-from-ingress
