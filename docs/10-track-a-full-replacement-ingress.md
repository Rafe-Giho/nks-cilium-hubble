# Track A. Cilium Full Replacement + Ingress + Hubble

- 목적: NKS 기본 Calico 클러스터를 Cilium primary CNI로 전환하고, Cilium Ingress와 Hubble Relay/UI/CLI를 검증합니다.
- 상태: draft
- 마지막 갱신: 2026-04-22

## 이 트랙의 위치

이 트랙이 현재 기본안입니다.

구성 목표:

- Calico VXLAN 기반 기존 CNI에서 Cilium primary CNI로 전환
- Hubble Relay/UI/CLI 활성화
- Cilium Ingress 활성화
- 샘플 HTTP 워크로드로 north-south/east-west 트래픽 확인

## 중단 조건

아래 조건 중 하나라도 해결되지 않으면 설치를 시작하지 않습니다.

- NKS에서 CNI 변경이 공식 비지원 PoC라는 점을 수용하지 못함
- 기존 Pod CIDR, Service CIDR, Node/VPC CIDR과 겹치지 않는 새 Cilium Pod CIDR을 확정하지 못함
- 노드 간 `TCP 4244` 허용 여부를 확인하지 못함
- PoC 실패 시 클러스터 재생성 롤백을 수용하지 못함

## 0. 실행 변수 지정

아래 값은 WSL 세션에서 지정합니다.

```bash
export CILIUM_VERSION="1.19.3"
export CILIUM_POD_CIDR="<NEW_DISTINCT_CILIUM_POD_CIDR>"
export CILIUM_TUNNEL_PORT="8473"
export DEMO_NAMESPACE="demo-a"
export DEMO_HOST="demo.example.local"
```

CIDR을 확정하지 않았다면 여기서 중단합니다.

```bash
test "$CILIUM_POD_CIDR" != "<NEW_DISTINCT_CILIUM_POD_CIDR>" || {
  echo "STOP: CILIUM_POD_CIDR is not set"
  exit 1
}
```

현재 NKS에서 관측된 Pod IP는 `10.100.184.0/24`, `10.100.188.0/24` 대역이고, Service IP는 `10.254.x.x` 대역입니다. 새 Cilium Pod CIDR은 이 대역들과 겹치면 안 됩니다.

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
```

기대 상태:

- 노드 3대 `Ready`
- NetworkPolicy 없음
- Ingress 없음
- LoadBalancer Service 없음
- Calico `calico_backend: vxlan`
- Calico image `v3.30.2`

## 2. Cilium secondary mode values 작성

이 단계는 Cilium을 바로 primary CNI로 만들지 않고, 별도 overlay를 먼저 구성하는 migration 시작점입니다.

```bash
cat > manifests/10-cilium-values-initial.yaml <<EOF
operator:
  unmanagedPodWatcher:
    restart: false
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
- `cni.customConf: true`
- `policyEnforcementMode: never`
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
```

기대 상태:

- Cilium agent/operator가 정상 기동
- 아직 기존 Pod 대부분은 Calico CNI 소속
- Hubble Relay/UI 리소스 생성

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

## 7. 전체 migration 확인

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

## 8. Cilium primary CNI + Ingress 최종 values 적용

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

## 9. Ingress 샘플 워크로드 배포

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

## 10. Hubble CLI/UI 확인

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

## 11. 후처리와 중단 기준

PoC 성공 기준:

- 모든 노드가 Cilium primary CNI로 동작
- Hubble Relay/UI/CLI 정상
- Cilium Ingress가 LoadBalancer를 받고 샘플 앱 호출 성공
- Hubble에서 HTTP/DNS 흐름 확인

중단 기준:

- 노드가 `Ready`로 복귀하지 않음
- CoreDNS 장애
- API server 접근 실패
- Cilium status 비정상
- Ingress LoadBalancer 생성 실패가 장시간 지속

Calico 삭제는 별도 판단이 필요합니다. NKS 관리형 애드온이 Calico를 재생성할 가능성이 있으므로, 이 문서에서는 자동 삭제 명령을 기본 절차에 넣지 않습니다.

## 참고 출처

- https://docs.cilium.io/en/stable/installation/k8s-install-migration/
- https://docs.cilium.io/en/stable/installation/k8s-install-helm/
- https://docs.cilium.io/en/stable/network/servicemesh/ingress/
- https://docs.cilium.io/en/stable/observability/hubble/setup/
