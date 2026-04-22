# 설치 가이드

- 목적: sidecar-less 트래픽 모니터링 목표에 맞는 구현 우선순위와 각 가이드 진입점을 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-04-17

## 현재 현황

- 이 저장소에는 전용 설치 가이드가 없었고, 관련 내용은 `reference/`와 검증 문서에만 흩어져 있었습니다.
- 현재 프로젝트 목표는 `sidecar-less 트래픽 모니터링`입니다.
- 현재 1순위 구현은 `Calico -> Cilium 전환`, `Cilium Ingress 활성화`, `Hubble Relay/UI/CLI` 구성입니다.
- 다만 NKS 릴리스 노트 기준으로 `cluster CNI change`는 비지원 상태이므로, 실제 적용은 `unsupported PoC`로 취급합니다.
- 사용자가 실제 명령을 실행하고, 이 문서는 그 기준 명령과 점검 포인트를 제공합니다.

## 관련 문서

- 범위/배경: `docs/project-scope.md`
- 검증 기준: `docs/validation-checklist.md`
- 샘플 트래픽 시나리오: `docs/poc-scenarios.md`
- Ingress vs Gateway API 비교: `docs/ingress-vs-gateway-api.md`
- Cilium Ingress 가이드: `docs/cilium-ingress-guide.md`
- Cilium Gateway API 가이드: `docs/cilium-gateway-api-guide.md`
- Calico chaining 가이드: `docs/cilium-calico-chaining-guide.md`
- Pixie 가이드: `docs/pixie-guide.md`
- Grafana Beyla 가이드: `docs/grafana-beyla-guide.md`
- 공식 근거 요약: `reference/official-findings.md`
- 출처 추적: `reference/source-tracking.md`

## 우선순위

1. `Cilium full replacement + Ingress + Hubble`
2. `Cilium full replacement + Gateway API + Hubble`
3. `Calico 유지 + Cilium chaining + Hubble`
4. `Pixie`
5. `Grafana Beyla`

## 권장 접근

현재 목적이 `sidecar-less 트래픽 모니터링`이라면 위 우선순위 기준으로 검토합니다.

### 1순위

- `docs/cilium-ingress-guide.md`
- 가장 빠르게 north-south와 Hubble 가시성을 한 번에 확인하기 좋습니다.

### 2순위

- `docs/cilium-gateway-api-guide.md`
- 장기 구조 관점에서는 더 유리하지만 1차 PoC로는 복잡도가 더 높습니다.

### 3순위

- `docs/cilium-calico-chaining-guide.md`
- 리스크가 낮고 Hubble 가시성만 빨리 보기에 좋지만 full replacement는 아닙니다.

### 추가 후보

- `docs/pixie-guide.md`
- `docs/grafana-beyla-guide.md`

둘 다 sidecar-less 관측 목적에는 부합하지만, Cilium처럼 CNI/Ingress/Gateway를 같이 풀어주지는 않습니다.

## 사전 준비

아래 명령은 WSL 기준입니다.

### 1. 기본 도구

```bash
kubectl version
helm version
```

### 2. Cilium CLI 설치

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all \
  https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

### 3. Hubble CLI 설치

```bash
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
HUBBLE_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi
curl -L --fail --remote-name-all \
  https://github.com/cilium/hubble/releases/download/${HUBBLE_VERSION}/hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-${HUBBLE_ARCH}.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz /usr/local/bin
rm hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
```

### 4. 설치 전 정보 수집

```bash
kubectl get no -o wide
kubectl get pods -A -o wide
kubectl get ds -A
kubectl get deploy -A
kubectl get svc -A
kubectl get ingress -A
kubectl get networkpolicy -A
```

## 설치 트랙 A: 현재 프로젝트 목표

Calico에서 Cilium으로 전환하는 트랙입니다. 현재 프로젝트 기본 방향은 이 트랙입니다.

### 핵심 주의점

- Cilium 공식 문서는 기존 CNI에서 Cilium으로의 마이그레이션 시 `새로운 Cluster CIDR`, `cluster-pool IPAM`, `기존 CNI와 다른 오버레이`를 요구합니다.
- 기존 NetworkPolicy provider가 있으면 제약이 있습니다.
- NKS는 공식적으로 cluster CNI change가 비지원 상태입니다.

즉, 이 트랙은 `설치 명령부터 바로 실행`보다 먼저 `현재 Pod CIDR`, `Service CIDR`, `Calico 구성`, `롤백 경로`를 확정해야 합니다.

### 실행 전 필수 확인

```bash
kubectl get no -o jsonpath='{range .items[*]}{.metadata.name}{"  podCIDR="}{.spec.podCIDR}{"\n"}{end}'
kubectl get svc kubernetes -o wide
kubectl get cm -n kube-system
kubectl get cm -n kube-system calico-config -o yaml
```

### Cilium 설치 방식

이 트랙은 아직 값 확정 전 단계이므로 아래 값을 먼저 고정해야 합니다.

- `CILIUM_VERSION`
- `NEW_POD_CIDR`
- `routingMode`
- `kubeProxyReplacement`
- `ingressController.loadbalancerMode`

값이 확정되면 Helm 기준 기본 골격은 아래 형태입니다.

```bash
helm upgrade --install cilium oci://quay.io/cilium/charts/cilium \
  --version <CILIUM_VERSION> \
  --namespace kube-system \
  --set ipam.mode=cluster-pool \
  --set ipam.operator.clusterPoolIPv4PodCIDRList={<NEW_POD_CIDR>} \
  --set kubeProxyReplacement=true \
  --set ingressController.enabled=true \
  --set ingressController.loadbalancerMode=dedicated \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true
```

이 명령은 `예시 골격`입니다. 현재 NKS용 최종 값은 아직 확정되지 않았으므로 그대로 실행하면 안 됩니다.

## 설치 트랙 B: 대안

Calico는 유지하고 Cilium chaining으로 Hubble 가시성부터 보는 트랙입니다.

### 언제 적합한가

- 우선 Hubble UI/CLI와 flow 가시성만 확인하고 싶을 때
- CNI 전체 전환 리스크를 늦추고 싶을 때
- 현재 단계가 `모니터링 환경 구축` 중심일 때

### 핵심 명령 골격

Calico chaining은 Cilium 공식 문서 기준으로 별도 `cni-configuration` ConfigMap과 Helm 설정이 필요합니다.

```bash
helm upgrade --install cilium oci://quay.io/cilium/charts/cilium \
  --version <CILIUM_VERSION> \
  --namespace kube-system \
  --set cni.chainingMode=generic-veth \
  --set cni.customConf=true \
  --set cni.configMap=cni-configuration \
  --set routingMode=native \
  --set enableIPv4Masquerade=false \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true
```

이 트랙도 Calico 설정과 MTU 확인 후 적용해야 합니다.

## Hubble Relay/CLI 확인

Hubble Relay 접근은 아래 둘 중 하나로 확인합니다.

### 방법 1. Cilium CLI 사용

```bash
cilium hubble port-forward
hubble status
hubble observe
```

### 방법 2. kubectl 사용

```bash
kubectl -n kube-system port-forward service/hubble-relay 4245:80
hubble status
hubble observe
```

필터 예시는 아래처럼 사용합니다.

```bash
hubble observe --namespace kube-system
hubble observe --pod kube-system/coredns-xxxxx
hubble observe --verdict DROPPED
```

## Hubble UI 접속

### 방법 1. Cilium CLI

```bash
cilium hubble ui
```

기본적으로 로컬 `12000` 포트로 포워딩됩니다. 브라우저가 자동으로 열리지 않으면 아래 주소를 엽니다.

```text
http://localhost:12000
```

### 방법 2. kubectl 포트포워드

먼저 Hubble UI 서비스 이름을 확인합니다.

```bash
kubectl -n kube-system get svc | grep hubble
```

일반적으로 아래처럼 접근합니다.

```bash
kubectl -n kube-system port-forward service/hubble-ui 12000:80
```

서비스 포트가 다르면 `kubectl -n kube-system get svc hubble-ui -o yaml`로 실제 포트를 먼저 확인합니다.

## Cilium Ingress 확인

설치 후에는 아래 자원을 확인합니다.

```bash
kubectl -n kube-system get pods
kubectl -n kube-system get svc
kubectl get ingress -A
cilium status
```

Ingress가 활성화되면 `kube-system`에 공유 또는 전용 LoadBalancer Service가 생길 수 있습니다.

## 설치 후 최소 검증

```bash
cilium status --wait
cilium connectivity test
kubectl -n kube-system get pods
kubectl -n kube-system get svc
kubectl -n kube-system get endpoints
```

샘플 트래픽 검증은 `docs/poc-scenarios.md`를 기준으로 진행합니다.

## 지금 바로 봐야 할 파일

지금 상태에서 바로 참고할 파일은 아래 순서가 적절합니다.

1. `docs/install-runbook.md`
2. `docs/cilium-ingress-guide.md`
3. `docs/cilium-gateway-api-guide.md`
4. `docs/cilium-calico-chaining-guide.md`
5. `docs/ingress-vs-gateway-api.md`

## 남은 작업

- 트랙 A를 계속 갈지, 트랙 B를 먼저 짧게 검증할지 결정
- 현재 클러스터의 Pod CIDR / Service CIDR 확정
- 최종 Helm values 초안 작성
- 샘플 워크로드 manifest 추가
