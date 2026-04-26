# Track B. Cilium Full Replacement + Gateway API + Hubble

- 목적: Cilium primary CNI 전환 후 Gateway API로 north-south 트래픽을 검증합니다.
- 상태: draft
- 마지막 갱신: 2026-04-22

## 이 트랙의 위치

이 트랙은 2순위입니다.

Ingress보다 장기 구조에는 유리하지만, CRD와 리소스 모델이 늘어나므로 1차 PoC 진입 속도는 낮습니다.

## 전제

이 트랙은 Cilium full replacement가 필요합니다. CNI migration 자체는 `docs/10-track-a-full-replacement-ingress.md`의 0~8단계를 그대로 사용합니다.

단, Track B에서는 최종 values 적용 시 Ingress 대신 Gateway API를 켭니다.

신규 노드 scale-out 안정화 전제도 Track A와 같습니다. NKS node group 기본 taint와 Cilium Helm `agentNotReadyTaintKey`가 일치해야 합니다.

## 0. 실행 변수 지정

```bash
export CILIUM_VERSION="1.19.3"
export DEMO_NAMESPACE="demo-a"
export DEMO_HOST="demo.example.local"
export CILIUM_AGENT_NOT_READY_TAINT_KEY="node.cilium.io/agent-not-ready"
```

## 1. CNI migration 수행

아직 Cilium primary CNI로 전환하지 않았다면 먼저 아래 문서의 0~8단계를 완료합니다.

```text
docs/10-track-a-full-replacement-ingress.md
```

완료 기준:

```bash
cilium status --wait
kubectl get no -o wide
kubectl get pod -A -o wide
```

## 2. Gateway API CRD 설치

Cilium stable 문서 기준 Gateway API `v1.4.1` CRD가 필요합니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_gateways.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_grpcroutes.yaml
```

TLSRoute까지 검증할 때만 추가합니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml
```

확인:

```bash
kubectl get crd | grep gateway.networking.k8s.io
```

## 3. Gateway API 최종 values 적용

Track A에서 생성한 initial values가 있다면 그것을 기반으로 Gateway API용 final values를 만듭니다.

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
  --set gatewayAPI.enabled=true \
  --set ingressController.enabled=false \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  > manifests/20-cilium-values-final-gateway-api.yaml
```

적용:

```bash
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --values manifests/20-cilium-values-final-gateway-api.yaml

kubectl -n kube-system rollout restart ds/cilium
kubectl -n kube-system rollout restart deploy/cilium-operator
cilium status --wait
```

확인:

```bash
kubectl get gatewayclass
kubectl get gateway -A
kubectl get httproute -A
kubectl -n kube-system get cm cilium-config -o yaml | grep -E 'agent-not-ready-taint-key|custom-cni-conf|cni-exclusive'
```

이후 `docs/10-track-a-full-replacement-ingress.md`의 `9-1. Per-node migration 설정 정리`와 `10. Scale-out 자동화 검증`을 동일하게 수행합니다.

## 4. 샘플 워크로드 배포

```bash
kubectl create namespace "${DEMO_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
```

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

## 5. Gateway/HTTPRoute 생성

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: demo-gateway
  namespace: ${DEMO_NAMESPACE}
spec:
  gatewayClassName: cilium
  listeners:
    - name: web
      port: 80
      protocol: HTTP
      hostname: ${DEMO_HOST}
      allowedRoutes:
        namespaces:
          from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-route
  namespace: ${DEMO_NAMESPACE}
spec:
  parentRefs:
    - name: demo-gateway
  hostnames:
    - ${DEMO_HOST}
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: demo-echo
          port: 80
EOF
```

상태 확인:

```bash
kubectl -n "${DEMO_NAMESPACE}" get gateway,httproute
kubectl -n "${DEMO_NAMESPACE}" describe gateway demo-gateway
kubectl -n "${DEMO_NAMESPACE}" describe httproute demo-route
kubectl get svc -A | grep -E 'cilium|LoadBalancer'
```

확인 기준:

- Gateway `Accepted=True`
- Gateway `Programmed=True`
- HTTPRoute가 Gateway에 attach
- LoadBalancer 주소 할당

## 6. 호출 테스트

```bash
export GATEWAY_ADDR="$(kubectl -n "${DEMO_NAMESPACE}" get gateway demo-gateway -o jsonpath='{.status.addresses[0].value}')"

curl -v -H "Host: ${DEMO_HOST}" "http://${GATEWAY_ADDR}/"
```

주소가 비어 있으면 Service에서 LoadBalancer 주소를 확인합니다.

```bash
kubectl get svc -A | grep LoadBalancer
```

## 7. Hubble 확인

```bash
cilium hubble port-forward
```

다른 터미널:

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

## 성공 기준

- Gateway API CRD 정상
- Cilium GatewayClass 생성
- Gateway/HTTPRoute `Accepted=True`, `Programmed=True`
- LoadBalancer 주소 할당
- 샘플 HTTP 호출 성공
- Hubble UI/CLI에서 Gateway -> backend 흐름 확인
- 신규 scale-out 노드가 Cilium primary CNI와 동일 taint 자동화 기준을 통과

## 참고 출처

- https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/
- https://gateway-api.sigs.k8s.io/concepts/api-overview/
