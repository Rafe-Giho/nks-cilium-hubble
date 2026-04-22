# 설치 런북 인덱스

- 목적: sidecar-less 트래픽 모니터링 PoC의 실행 순서와 각 가이드 진입점을 고정합니다.
- 상태: draft
- 마지막 갱신: 2026-04-22

## 기본 방향

현재 기본 방안은 `Track A: Cilium full replacement + Ingress + Hubble`입니다.

NKS 릴리스 노트 기준 cluster CNI change는 비지원 상태이므로, 모든 full replacement 절차는 운영 전환 절차가 아니라 `unsupported PoC` 절차로 취급합니다.

## 실행 원칙

- 클러스터 변경 명령은 사용자가 직접 실행합니다.
- 이 저장소는 실행 순서, 명령 예시, 검증 기준, 결과 기록을 제공합니다.
- 명령은 WSL 기준입니다.
- `STOP` 블록이 있으면 그 조건을 해결하기 전 다음 단계로 넘어가지 않습니다.
- 실패 시 복구 우선순위는 `PoC 클러스터 재생성`입니다.

## 가이드 순서

| 순서 | 파일 | 용도 |
| --- | --- | --- |
| 00 | `docs/00-install-runbook.md` | 전체 인덱스 |
| 05 | `docs/05-ingress-vs-gateway-api.md` | Ingress/Gateway API 선택 기준 |
| 10 | `docs/10-track-a-full-replacement-ingress.md` | 1순위 기본안: Cilium full replacement + Ingress + Hubble |
| 20 | `docs/20-track-b-full-replacement-gateway-api.md` | 2순위: Cilium full replacement + Gateway API + Hubble |
| 30 | `docs/30-track-c-calico-chaining-hubble.md` | 3순위: Calico 유지 + Cilium chaining + Hubble |
| 40 | `docs/40-option-pixie.md` | 추가 후보: Pixie |
| 50 | `docs/50-option-grafana-beyla.md` | 추가 후보: Grafana Beyla |

## 현재 클러스터 기준

현재 값은 `docs/environment-inventory.md`를 기준으로 합니다.

- Kubernetes server: `v1.33.4`
- Worker: 3대
- OS: `Ubuntu 24.04.4 LTS`
- Kernel: `6.8.0-88-generic`
- Runtime: `containerd://1.6.21`
- 현재 CNI: Calico VXLAN `v3.30.2`
- 현재 Ingress/NetworkPolicy/LoadBalancer Service: 없음

## CLI 설치 후 남은 파일

아래 파일은 Cilium CLI/Hubble CLI 설치용 압축 파일과 checksum입니다. 바이너리가 `/usr/local/bin` 등에 정상 설치되어 있으면 삭제해도 됩니다.

```text
cilium-linux-amd64.tar.gz
cilium-linux-amd64.tar.gz.sha256sum
hubble-linux-amd64.tar.gz
hubble-linux-amd64.tar.gz.sha256sum
```

삭제 전 확인:

```bash
cilium version --client
hubble version
```

삭제 명령:

```bash
rm -f cilium-linux-amd64.tar.gz cilium-linux-amd64.tar.gz.sha256sum
rm -f hubble-linux-amd64.tar.gz hubble-linux-amd64.tar.gz.sha256sum
```

## 공통 사전 확인

```bash
kubectl config current-context
kubectl get no -o wide
kubectl get pod -A -o wide
kubectl get svc -A
kubectl get ingress -A
kubectl get networkpolicy -A
helm version
cilium version --client
hubble version
```

기대 상태:

- 노드 3대가 `Ready`
- `NetworkPolicy` 없음
- `Ingress` 없음
- `LoadBalancer` Service 없음
- Calico 관련 Pod가 모두 `Running`

## 기본 실행 순서

1. `docs/environment-inventory.md`에서 확인 불가 항목을 확인합니다.
2. `docs/05-ingress-vs-gateway-api.md`로 Ingress/Gateway API 차이를 확인합니다.
3. 기본안이면 `docs/10-track-a-full-replacement-ingress.md`를 처음부터 실행합니다.
4. Gateway API를 보려면 `docs/20-track-b-full-replacement-gateway-api.md`를 실행합니다.
5. full replacement 리스크가 높다고 판단하면 `docs/30-track-c-calico-chaining-hubble.md`를 검토합니다.

## 설치 전 반드시 확인할 값

| 항목 | 현재 상태 | 필요 조치 |
| --- | --- | --- |
| 새 Cilium Pod CIDR | 미확정 | 기존 Pod/Service/Node/VPC CIDR과 겹치지 않는 CIDR 확정 |
| Service CIDR 공식값 | 미확정 | NKS 콘솔 또는 생성 파라미터에서 확인 |
| Hubble Relay `TCP 4244` | 미확정 | 노드 간 보안그룹 허용 확인 |
| NKS LoadBalancer 생성 | 미검증 | 샘플 Service/Ingress로 확인 |
| rollback | PoC 클러스터 재생성 | 실행 전 합의 |

## 공식 근거

- Cilium Helm 설치: https://docs.cilium.io/en/stable/installation/k8s-install-helm/
- Cilium migration: https://docs.cilium.io/en/stable/installation/k8s-install-migration/
- Cilium Ingress: https://docs.cilium.io/en/stable/network/servicemesh/ingress/
- Cilium Gateway API: https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/
- Hubble: https://docs.cilium.io/en/stable/observability/hubble/setup/
