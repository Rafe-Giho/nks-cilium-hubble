# NKS Cilium PoC 문서 인덱스

- 목적: sidecar-less 트래픽 모니터링 PoC의 실행 순서와 각 가이드 진입점을 고정합니다.
- 상태: draft
- 마지막 갱신: 2026-05-14

## 기본 방향

`Track A: Cilium full replacement`는 2026-04-30 기준 실패로 종결했습니다.

NKS 릴리스 노트 기준 cluster CNI change는 비지원 상태이며, 실제 실험에서도 Calico VXLAN 호환 경로 없이는 control-plane-to-pod 경로가 정상화되지 않았습니다.

현재 `90`, `91` 문서는 full replacement 실패 기록 보존용입니다. 후속 실행 가이드는 Calico 유지 기반 Cilium chaining + Hubble과 Cilium Service Mesh 별도 트랙으로 관리합니다.

## 실행 원칙

- 클러스터 변경 명령은 사용자가 직접 실행합니다.
- 이 저장소는 실행 순서, 명령 예시, 검증 기준, 결과 기록을 제공합니다.
- 현재 실행 명령은 Ubuntu Linux 또는 WSL bash 기준입니다.
- kubeconfig는 `/mnt/c/Users/user/Downloads/ta-sgh-vxlan-sidecarless-cls_kubeconfig.yaml`을 사용합니다.
- `STOP` 블록이 있으면 그 조건을 해결하기 전 다음 단계로 넘어가지 않습니다.
- 실패 시 복구 우선순위는 `PoC 클러스터 재생성`입니다.

## 가이드 순서

| 순서 | 파일 | 용도 |
| --- | --- | --- |
| 00 | `docs/00-install-runbook.md` | 전체 인덱스 |
| 01 | `docs/01-observability-architecture-concepts.md` | Calico chaining, Cilium, Hubble, Grafana, exporter 개념과 연계 구조 |
| 02 | `docs/02-ingress-vs-gateway-api-reference.md` | full replacement 시도 당시 Ingress/Gateway API 참고 자료 |
| 03 | `docs/03-environment-inventory.md` | 신규 NKS 클러스터 환경 기준선 |
| 04 | `docs/04-nks-security-group.md` | NKS 기본 보안그룹과 Cilium/Hubble 필요 포트 확인 |
| 05 | `docs/05-calico-chaining-hubble-runbook.md` | Calico 유지 + Cilium chaining + Hubble 실행 가이드 |
| 06 | `docs/06-hubble-observability-runbook.md` | Hubble UI/CLI, Grafana metrics, exporter 기반 모니터링 가이드 |
| SM | `docs/service-mesh/` | 고객 요청 시 Cilium Gateway API/GAMMA 기반 Service Mesh 별도 트랙 |
| 90 | `docs/90-full-replacement-execution-record.md` | NKS Cilium full replacement 실행 기록 아카이브 |
| 91 | `docs/91-full-replacement-assets-archive.md` | NKS Cilium full replacement manifest/script 자산 아카이브 |

## 현재 클러스터 기준

현재 값은 `docs/03-environment-inventory.md`를 기준으로 합니다.

- Cluster: `ta-sgh-vxlan-sidecarless-cls`
- Kubernetes server: `v1.33.4`
- Worker: 2대
- OS: `Ubuntu 22.04.5 LTS`
- Kernel: `5.15.0-164-generic`
- Runtime: `containerd://1.6.21`
- 현재 CNI: NKS 기본 Calico `v3.30.2`, `calico_backend=vxlan`, `DATASTORE_TYPE=etcdv3`
- 현재 Ingress/NetworkPolicy/LoadBalancer Service: 없음
- metrics APIService: 정상
- Helm release: `kube-system/cilium`
- Gateway API CRD: 없음

## CLI 설치 후 남은 파일

아래 파일은 Cilium CLI/Hubble CLI 설치용 압축 파일과 checksum입니다. 바이너리가 정상 설치되어 있으면 삭제해도 됩니다.

```text
cilium-linux-amd64.tar.gz
cilium-linux-amd64.tar.gz.sha256sum
hubble-linux-amd64.tar.gz
hubble-linux-amd64.tar.gz.sha256sum
```

확인:

```bash
cilium version --client
hubble version
```

## 공통 사전 확인

```bash
export KUBECONFIG="/mnt/c/Users/user/Downloads/ta-sgh-vxlan-sidecarless-cls_kubeconfig.yaml"

kubectl config current-context
kubectl get no -o wide
kubectl get pod -A -o wide
kubectl get svc -A
kubectl get ingress -A
kubectl get networkpolicy -A
kubectl get apiservice v1beta1.metrics.k8s.io -o wide
helm version
cilium version --client
hubble version
```

기대 상태:

- 노드가 `Ready`
- `calico-node`가 모든 노드에서 Ready
- `NetworkPolicy` 없음
- `Ingress` 없음
- `LoadBalancer` Service 없음
- metrics APIService 정상
- 현재 클러스터는 Calico VXLAN + Cilium chaining/Hubble 기준선

## 기본 실행 순서

1. `docs/01-observability-architecture-concepts.md`에서 현재 구조와 도구 역할을 확인합니다.
2. `docs/03-environment-inventory.md`에서 신규 클러스터 기준선을 확인합니다.
3. `docs/04-nks-security-group.md`에서 보안그룹 전제와 포트를 확인합니다.
4. Cilium 기반 검증은 `docs/05-calico-chaining-hubble-runbook.md`를 우선 실행합니다.
5. Hubble을 모니터링 용도로 확장하려면 `docs/06-hubble-observability-runbook.md`를 확인합니다.
6. 고객 요청으로 Service Mesh 도입이 필요하면 `docs/service-mesh/`와 `manifests/service-mesh/`를 별도 트랙으로 검토합니다.
7. Ingress/Gateway API 문서(`docs/02-ingress-vs-gateway-api-reference.md`)와 full replacement 실패 기록(`docs/90-full-replacement-execution-record.md`, `docs/91-full-replacement-assets-archive.md`)은 참고/아카이브로 봅니다.

운영형 Grafana 검증은 `docs/06-hubble-observability-runbook.md` 3장을 따릅니다. 적용 자산은 `manifests/06-kube-prometheus-stack-values.yaml`과 `manifests/06-cilium-hubble-metrics-values.yaml`입니다.

## 설치 전 반드시 확인할 값

| 항목 | 현재 상태 | 필요 조치 |
| --- | --- | --- |
| 새 Cilium Pod CIDR | 05번 실행 트랙에서는 사용 안 함 | Calico IPAM 유지 |
| Service CIDR 공식값 | 미확정 | NKS 콘솔 또는 생성 파라미터에서 확인 |
| Calico VXLAN `UDP 4789` | 사용 중 | 유지 |
| Hubble Relay `TCP 4244` | 충족 예상 | 현재 NKS worker self `TCP 1-65535` 유지 확인 |
| Cilium health `TCP 4240`/ICMP | 일부 충족 예상 | `TCP 4240`은 worker self TCP 전체로 충족, ICMP는 필요 시 추가 검토 |
| NKS Calico CNI 경로 | 확인됨 | `/etc/cni/net.d/calico` |
| NKS LoadBalancer 생성 | 미검증 | 샘플 Service/Ingress로 확인 |
| rollback | PoC 클러스터 재생성 | 실행 전 합의 |

## 공식 근거

- Cilium Helm 설치: https://docs.cilium.io/en/stable/installation/k8s-install-helm/
- Cilium Calico chaining: https://docs.cilium.io/en/stable/installation/cni-chaining-calico/
- Cilium generic-veth chaining: https://docs.cilium.io/en/stable/installation/cni-chaining-generic-veth.html
- Cilium firewall rules: https://docs.cilium.io/en/stable/operations/system_requirements/
- Calico network requirements: https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements
- Hubble: https://docs.cilium.io/en/stable/observability/hubble/setup/
- Hubble UI: https://docs.cilium.io/en/stable/observability/hubble/hubble-ui/
- Hubble metrics/Grafana: https://docs.cilium.io/en/stable/observability/grafana/
- Hubble exporter: https://docs.cilium.io/en/stable/observability/hubble/configuration/export/
