# Pixie 가이드

- 목적: Cilium 외 대안으로서 Pixie를 사용해 sidecar-less Kubernetes 트래픽/애플리케이션 관측을 수행하는 절차를 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-04-17

## 왜 후보인가

- Pixie는 eBPF 기반으로 수동 계측 없이 Kubernetes 애플리케이션 관측을 제공합니다.
- 네트워크 흐름, DNS, 서비스 맵, 요청 추적, 프로파일링까지 폭이 넓습니다.
- sidecar-less 관측 목적에는 잘 맞습니다.

## 이 방식의 위치

- 현재 프로젝트의 `추가 후보 1`입니다.
- 이유는 네트워크 관측 범위는 넓지만, CNI/Ingress를 대체하지는 않기 때문입니다.

## 핵심 특징

- eBPF 기반 auto-telemetry
- CLI, Live UI, API 제공
- 네트워크 모니터링, DNS, 서비스 성능, 요청 추적 등 제공
- 클러스터 내부에 Vizier가 설치되고, 운영 방식에 따라 Pixie Cloud와 연동할 수 있습니다.

## 주의점

- Pixie는 CNI replacement가 아닙니다.
- Cilium Ingress/Gateway API 같은 north-south 진입점 제어를 대신하지 않습니다.
- 기본 Hosted 설치는 외부 Pixie Cloud와의 통신이 필요합니다.
- OLM과 privileged access 요구사항이 있습니다.

## 공식 요구사항 핵심

- Kubernetes `1.21+`
- Linux kernel `4.14+`
- Ubuntu `18.04+` 계열 지원 목록 포함
- 노드당 최소 메모리 `1GiB`, 기본 권장 메모리 제한은 `2Gi`
- Pixie Vizier는 Pixie Cloud로 `443` 포트 아웃바운드가 필요할 수 있음
- PEM Pod는 privileged access가 필요

## 설치 방식

Pixie 공식 문서는 `CLI` 방식을 가장 쉬운 방법으로 권장합니다.

## 1. Pixie CLI 설치

```bash
bash -c "$(curl -fsSL https://withpixie.ai/install.sh)"
```

또는 직접 바이너리 방식:

```bash
curl -o px https://storage.googleapis.com/pixie-dev-public/cli/latest/cli_linux_amd64
chmod +x px
sudo mv px /usr/local/bin
```

## 2. 요구사항 점검

```bash
export PX_CLOUD_ADDR=getcosmic.ai
px deploy --check_only
```

## 3. Pixie 배포

OLM이 없는 경우:

```bash
export PX_CLOUD_ADDR=getcosmic.ai
px deploy
```

OLM이 이미 있는 경우:

```bash
export PX_CLOUD_ADDR=getcosmic.ai
px deploy --deploy_olm=false
```

메모리 제한을 줄이려면:

```bash
export PX_CLOUD_ADDR=getcosmic.ai
px deploy --pem_memory_limit=1Gi
```

## 4. 기본 상태 확인

```bash
px get viziers
px get pems
kubectl get ns
kubectl get pods -n px-operator
kubectl get pods -n pl
```

## 5. 네트워크 모니터링 예시

CLI에서 스크립트 목록 확인:

```bash
px run -l
```

네트워크 흐름 관련 예시는 Live UI가 더 편합니다.

- `px/net_flow_graph`
- DNS 흐름
- TCP drops / retransmits 확인

CLI 예시는:

```bash
px run px/namespaces
```

## 6. Live UI 사용

Pixie 공식 문서는 처음에는 Live UI 사용을 권장합니다.

- 네트워크 흐름 그래프
- 서비스 간 통신
- DNS 요청 흐름
- TCP drop/retransmit

## 이 방식의 장점

- 아주 넓은 관측 범위를 제공합니다.
- 네트워크뿐 아니라 요청/프로파일링까지 확장됩니다.
- sidecar-less라는 목적에는 잘 맞습니다.

## 이 방식의 한계

- Cilium처럼 CNI/Ingress/Gateway를 한 번에 다루지 않습니다.
- 운영 모델상 Pixie Cloud 또는 self-hosted Pixie Cloud 고려가 필요합니다.
- 단순 `Hubble만 보기`보다 구성 요소가 큽니다.

## 추천 사용 시점

- 네트워크뿐 아니라 애플리케이션 레벨 가시성까지 같이 보고 싶을 때
- 서비스 맵과 요청 분석까지 한 번에 보고 싶을 때
- CNI 교체 없이 강한 관측 플랫폼을 붙이고 싶을 때

## 참고 출처

- https://docs.px.dev/about-pixie/what-is-pixie/
- https://docs.px.dev/installing-pixie/requirements/
- https://docs.px.dev/installing-pixie/install-schemes/cli/
- https://docs.px.dev/tutorials/pixie-101/network-monitoring/
- https://docs.px.dev/using-pixie/using-cli/
