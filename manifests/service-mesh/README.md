# Cilium Service Mesh manifests

- 목적: Cilium Service Mesh PoC용 Helm values, Gateway API CRD kustomization, 샘플 리소스를 관리합니다.
- 상태: draft
- 마지막 갱신: 2026-05-13

## 파일 목록

| 경로 | 용도 |
| --- | --- |
| `10-cilium-gateway-api-values.yaml` | 현재 chaining/Hubble 설정을 유지하면서 Gateway API, kube-proxy replacement, EnvoyConfig, L7 proxy를 활성화했던 검증 values |
| `11-cilium-ingress-optional-values.yaml` | Ingress controller까지 검증할 때 추가로 사용하는 overlay values |
| `12-cilium-gateway-hostnetwork-overlay-values.yaml` | LoadBalancer 미사용 시 hostNetwork Gateway 대안 overlay values |
| `13-cilium-gateway-hostnetwork-no-original-source-values.yaml` | hostNetwork 검증 중 원본 소스 주소 사용을 끄는 보정 overlay values |
| `14-cilium-ingress-hostnetwork-values.yaml` | Cilium Ingress hostNetwork 대안 검증 values |
| `gateway-api-crds-standard/kustomization.yaml` | Gateway API v1.4.1 `standard-install.yaml` 공식 bundle 참조 |
| `gateway-api-crds-experimental/kustomization.yaml` | Gateway API v1.4.1 `experimental-install.yaml` 공식 bundle 참조 |
| `20-demo-gateway-http.yaml` | Gateway API north-south HTTP 샘플 |
| `21-demo-gamma-http-route.yaml` | GAMMA Service parentRef HTTPRoute 샘플 |
| `22-demo-gateway-http-hostnetwork.yaml` | hostNetwork Gateway 샘플 |
| `23-demo-gateway-http-nodeport.yaml` | NodePort GatewayClass 샘플 |
| `24-debug-hostnetwork-curl.yaml` | hostNetwork 경로 확인용 curl Pod |
| `25-demo-cilium-ingress-hostnetwork.yaml` | Cilium Ingress hostNetwork 샘플 |
| `30-cilium-primary-service-mesh-values.yaml` | Cilium primary CNI 환경용 Service Mesh + Hubble L7 values 샘플 |
| `31-demo-cilium-l7-policy.yaml` | Cilium HTTP L7 proxy 동작 검증용 CiliumNetworkPolicy 샘플 |

## 적용 원칙

- 이 디렉터리의 파일은 초안입니다.
- 현재 NKS Calico chaining 클러스터에는 `10~25` 검증 자산을 운영 적용하지 않습니다.
- Cilium primary CNI가 가능한 별도 환경에서만 `docs/service-mesh/01-cilium-service-mesh-runbook.md`의 dry-run과 중단 조건을 확인합니다.
- `kubeProxyReplacement=true`, CRD 설치, LoadBalancer/NodePort/hostNetwork 노출은 영향 범위가 큰 변경입니다.
- Gateway API CRD는 직접 작성하지 않고 공식 release bundle URL을 고정해 추적합니다.
- 현재 NKS Calico chaining 구성에서의 성공/실패 판정은 `docs/service-mesh/05-build-result-report.md`에 기록합니다.
