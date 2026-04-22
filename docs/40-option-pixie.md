# Option 1. Pixie

- 목적: Cilium 외 sidecar-less 관측 후보로 Pixie 실행 절차와 한계를 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-04-22

## 이 옵션의 위치

Pixie는 CNI replacement가 아닙니다. Cilium Ingress/Gateway를 대체하지도 않습니다.

다만 eBPF 기반으로 네트워크 흐름, DNS, 서비스 맵, 요청 관측을 폭넓게 제공하므로 sidecar-less 관측 후보로는 유효합니다.

## 선택 조건

Pixie가 맞는 경우:

- CNI 변경 없이 관측을 먼저 보고 싶음
- Hubble보다 더 넓은 앱/서비스 관측을 원함
- Pixie Cloud 또는 self-hosted Pixie Cloud 운영을 검토할 수 있음

Pixie가 맞지 않는 경우:

- Cilium CNI/Ingress/Gateway 전환 검증이 목표
- 외부 SaaS 연동이 불가
- privileged agent 설치가 불가

## 0. 요구사항 확인

```bash
kubectl version
kubectl get no -o wide
kubectl auth can-i create pods --all-namespaces
kubectl auth can-i create clusterrole
```

확인할 것:

- Kubernetes `1.21+`
- Linux kernel `4.14+`
- outbound `443` 가능 여부
- privileged Pod 허용 여부

## 1. Pixie CLI 설치

```bash
bash -c "$(curl -fsSL https://withpixie.ai/install.sh)"
```

또는 직접 설치:

```bash
curl -o px https://storage.googleapis.com/pixie-dev-public/cli/latest/cli_linux_amd64
chmod +x px
sudo mv px /usr/local/bin
```

확인:

```bash
px version
```

## 2. 설치 가능성 점검

```bash
export PX_CLOUD_ADDR=getcosmic.ai
px deploy --check_only
```

실패하면 요구사항 또는 outbound 정책을 먼저 해결합니다.

## 3. Pixie 배포

OLM이 없는 일반 상황:

```bash
export PX_CLOUD_ADDR=getcosmic.ai
px deploy
```

OLM이 이미 있는 경우:

```bash
export PX_CLOUD_ADDR=getcosmic.ai
px deploy --deploy_olm=false
```

메모리 제한을 낮춰 검증할 경우:

```bash
export PX_CLOUD_ADDR=getcosmic.ai
px deploy --pem_memory_limit=1Gi
```

## 4. 상태 확인

```bash
px get viziers
px get pems
kubectl get pods -n px-operator
kubectl get pods -n pl
```

## 5. 네트워크 관측

스크립트 목록:

```bash
px run -l
```

기본 확인:

```bash
px run px/namespaces
```

Live UI에서 볼 항목:

- network flow graph
- DNS 흐름
- 서비스 간 통신
- TCP drop/retransmit

## 성공 기준

- Vizier 정상
- PEM 정상
- Live UI 접속 가능
- namespace/service 흐름 확인 가능

## 한계

- Cilium/Hubble 대체가 아니라 별도 관측 플랫폼입니다.
- CNI, Ingress, Gateway API 기능을 제공하지 않습니다.
- 외부 Pixie Cloud 또는 self-hosted 운영 모델 검토가 필요합니다.

## 참고 출처

- https://docs.px.dev/about-pixie/what-is-pixie/
- https://docs.px.dev/installing-pixie/requirements/
- https://docs.px.dev/installing-pixie/install-schemes/cli/
- https://docs.px.dev/tutorials/pixie-101/network-monitoring/
