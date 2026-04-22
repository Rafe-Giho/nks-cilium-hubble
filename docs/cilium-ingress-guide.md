# Cilium Ingress 가이드

- 목적: Cilium이 설치된 클러스터에서 Cilium Ingress를 활성화하고 검증하는 절차를 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-04-17

## 언제 이 방식을 쓰나

- 가장 빠르게 north-south HTTP 경로를 띄우고 싶을 때
- Gateway API보다 단순한 리소스 모델이 필요할 때
- 1차 PoC에서 Hubble과 함께 기본 Ingress 흐름만 보고 싶을 때

## 전제 조건

- Cilium이 먼저 설치되어 있어야 합니다.
- `kubeProxyReplacement=true`가 필요합니다.
- `l7Proxy=true`가 필요합니다.
- 기본 노출 방식은 `LoadBalancer`입니다.
- LoadBalancer를 쓰기 어렵다면 `NodePort` 또는 `hostNetwork`를 검토합니다.

## 활성화 명령

기존 Cilium 릴리스가 있다고 가정한 Helm 기준 명령입니다.

```bash
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --version 1.19.3 \
  --namespace kube-system \
  --reuse-values \
  --set kubeProxyReplacement=true \
  --set ingressController.enabled=true \
  --set ingressController.loadbalancerMode=dedicated

kubectl -n kube-system rollout restart deployment/cilium-operator
kubectl -n kube-system rollout restart ds/cilium
```

### 모드 선택

- `dedicated`
  각 Ingress마다 별도 LB를 가집니다.
  PoC에서 충돌이 적고 해석이 쉽습니다.
- `shared`
  모든 Ingress가 공유 LB를 사용합니다.
  자원은 덜 쓰지만 라우팅 충돌과 해석 복잡도가 올라갈 수 있습니다.

현재 PoC는 `dedicated`를 기본 권장값으로 둡니다.

## 기본 확인 명령

```bash
cilium status
kubectl -n kube-system get pods
kubectl -n kube-system get svc
kubectl get ingress -A
```

## 샘플 Ingress 리소스

아래는 샘플 HTTP backend가 이미 있다고 가정한 최소 예시입니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  namespace: demo-a
  annotations:
    ingress.cilium.io/loadbalancer-mode: dedicated
spec:
  ingressClassName: cilium
  rules:
    - host: demo.example.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: demo-echo
                port:
                  number: 80
```

## Hubble 관점 확인

Ingress가 활성화된 뒤 아래 순서로 확인합니다.

```bash
cilium hubble port-forward
hubble status
hubble observe --namespace demo-a
```

Ingress 호출 트래픽을 발생시킨 뒤 Hubble UI 또는 CLI에서 아래를 봅니다.

- 외부 요청이 Envoy까지 들어오는지
- Envoy에서 backend Service/Pod로 전달되는지
- drop event가 없는지

## Hubble UI 접속

```bash
cilium hubble ui
```

브라우저가 자동으로 열리지 않으면 아래 주소를 엽니다.

```text
http://localhost:12000
```

## 자주 쓰는 Ingress annotation

- `ingress.cilium.io/loadbalancer-mode`
- `ingress.cilium.io/service-type`
- `ingress.cilium.io/service-external-traffic-policy`
- `ingress.cilium.io/host-listener-port`
- `ingress.cilium.io/tls-passthrough`
- `ingress.cilium.io/force-https`

## hostNetwork 대안

LoadBalancer 없이 외부 LB나 노드 직접 접근을 쓸 때는 host network 모드를 검토할 수 있습니다.

```yaml
ingressController:
  enabled: true
  hostNetwork:
    enabled: true
```

주의점은 아래와 같습니다.

- LoadBalancer/NodePort 방식과 상호 배타적입니다.
- 포트 충돌을 직접 관리해야 합니다.
- `dedicated` 모드에서 여러 Ingress를 쓰면 host listener 포트 충돌 가능성이 있습니다.

## 이 방식의 장점

- 빠르게 붙습니다.
- 리소스 수가 적습니다.
- 샘플 워크로드로 검증하기 쉽습니다.

## 이 방식의 한계

- Gateway API보다 구조적 표현력이 낮습니다.
- 고급 라우팅과 역할 분리는 Gateway API가 더 자연스럽습니다.
- 장기 표준으로 가져갈지 여부는 별도 검토가 필요합니다.

## 추천 사용 시점

- 첫 north-south 가시성 검증
- Hubble UI 연동 첫 확인
- 가장 단순한 PoC 기준선 수립

## 참고 출처

- https://docs.cilium.io/en/stable/network/servicemesh/ingress/
- https://docs.cilium.io/en/stable/helm-reference/
